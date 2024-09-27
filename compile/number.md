---
marp: true
theme: default
class: invert
paginate: true
---

# Compiling a Number to Bytecode

---

Suppose:

```lisp
(compile 11)
```

Let's trace how we end up with the corresponding Janet bytecode.

---

A peek at the execution path...

```
cfun_compile
  janet_compile_lint
    janetc_value
      janetc_cslot
      janetc_return
        janetc_emit_s
          janetc_regfar
            janetc_movenear
              janetc_loadconst
          janetc_emit
    janetc_pop_funcdef
```

---

```c
/* C Function for compiling */
JANET_CORE_FN(cfun_compile,
              "(compile ast &opt env source lints)",
              "Compiles an Abstract Syntax Tree (ast) into a function. "
              "Pair the compile function with parsing functionality to implement "
              "eval. Returns a new function and does not modify ast. Returns an error "
              // ...
              "`(level line col message)`.") {
    // ...
    JanetCompileResult res = janet_compile_lint(argv[0], env, source, lints);
                             //////////////////               //////
    // ...
```

The Janet function `compile` is implemented in C by `cfun_compile` in `compile.c`.

Among other things, `cfun_compile` calls `janet_compile_lint`, passing a janet value that wraps `11` via the parameter `source`.

---

```c
/* Compile a form. */
JanetCompileResult janet_compile_lint(Janet source,
                                      JanetTable *env, const uint8_t *where, JanetArray *lints) {
    JanetCompiler c;
    // ...
    janetc_init(&c, env, where, lints);
    // ...
    /* Set initial form options */
    fopts.compiler = &c;
    fopts.flags = JANET_FOPTS_TAIL | JANET_SLOTTYPE_ANY;
                  ////////////////
    // ...
    /* Compile the value */
    janetc_value(fopts, source);
    ////////////        //////

    if (c.result.status == JANET_COMPILE_OK) {
        JanetFuncDef *def = janetc_pop_funcdef(&c);
                            //////////////////
    // ...
```

After some preparation, `janet_compile_lint` calls `janetc_value`, passing our wrapped `11` as `source`.  Note that during the preparation, `fopts.flags` comes to include `JANET_FOPTS_TAIL`; this will be relevant later.

If the compilation is successful, `janet_compile_lint` calls `janetc_pop_funcdef`, and we'll return to this subsequently.

---

```c
/* Compile a single value */
JanetSlot janetc_value(JanetFopts opts, Janet x) {
    JanetSlot ret;
    JanetCompiler *c = opts.compiler;
    // ...
        switch (janet_type(x)) {
            case JANET_TUPLE: {
            // ...
            case JANET_SYMBOL:
            // ...
            case JANET_ARRAY:
            // ...
            case JANET_STRUCT:
            // ...
            case JANET_TABLE:
            // ...
            case JANET_BUFFER:
            // ...
            default:
                ret = janetc_cslot(x);
                      ////////////
    // ...
    if (opts.flags & JANET_FOPTS_TAIL)
        ret = janetc_return(c, ret);
              /////////////
    // ...
    return ret;
}
```

As `x` is a wrapped janet number, the switch's default case is entered and hence, `janetc_cslot` is called.  Since `JANET_FOPTS_TAIL` is set in `opts.flags`, `janetc_return` is called.  Let's visit `janetc_cslot` first.

---

```c
/* Create a slot with a constant */
JanetSlot janetc_cslot(Janet x) {
    JanetSlot ret;
    ret.flags = (1 << janet_type(x)) | JANET_SLOT_CONSTANT;
        /////                          ///////////////////
    ret.index = -1;
        /////   //
    ret.constant = x;
        ////////   //
    ret.envindex = -1;
    return ret;
}
```

`janetc_cslot` produces a `JanetSlot` for a constant value.

Note that the slot's `flags` member includes `JANET_SLOT_CONSTANT`, the `index` member is `-1`, and the `constant` member is the wrapped `11` value.

Shortly after `janetc_cslot` returns to `janetc_value`, `janetc_return` is called.

---

```c
/* Generate the return instruction for a slot. */
JanetSlot janetc_return(JanetCompiler *c, JanetSlot s) {
    if (!(s.flags & JANET_SLOT_RETURNED)) {
        if (s.flags & JANET_SLOT_CONSTANT && janet_checktype(s.constant, JANET_NIL))
            janetc_emit(c, JOP_RETURN_NIL);
        else
            janetc_emit_s(c, JOP_RETURN, s, 0);
            /////////////    //////////
        s.flags |= JANET_SLOT_RETURNED;
                   ///////////////////
    }
    return s;
}
```

In our case `s.constant` is not `nil` so `janetc_emit_s` is called, being passed `JOP_RETURN`.

Note that subsequently, `s.flags` will end up including `JANET_SLOT_RETURNED`.  This appears to be an indicator that `s` has been used as part of a `JOP_RETURN*`(?).

---

```c
int32_t janetc_emit_s(JanetCompiler *c, uint8_t op, JanetSlot s, int wr) {
    int32_t reg = janetc_regfar(c, s, JANETC_REGTEMP_0);
                  /////////////
    // ...
    janetc_emit(c, op | (reg << 8));
    ///////////    //
    // ...
```

The primary action of `janetc_emit_s` is to call `janetc_emit`.  In this case the call is made with opcode `op` (`JOP_PUSH`) and `s`, a `JanetSlot`, for our wrapped value `11`.

But before calling `janetc_emit`, `janetc_emit_s` calls `janetc_regfar` (passing it `s` for our wrapped value `11"`).  Time for a bit of a detour...

---

```c
/* Convert a slot to a two byte register */
static int32_t janetc_regfar(JanetCompiler *c, JanetSlot s, JanetcRegisterTemp tag) {
    /* check if already near register */
    if (s.envindex < 0 && s.index >= 0) {
                          ////////////
        return s.index;
    }
    int32_t reg;
    int32_t nearreg = janetc_regalloc_temp(&c->scope->ra, tag);
    janetc_movenear(c, nearreg, s);
    ///////////////
    // ...
```

In `janetc_regfar`, since `s` was made via `janetc_cslot`, `s.index` is `-1`.  Therefore, the body of the `if` is skipped and `janetc_movenear` will be called.

---

```c
/* Move a slot to a near register */
static void janetc_movenear(JanetCompiler *c,
                            int32_t dest,
                            JanetSlot src) {
    if (src.flags & (JANET_SLOT_CONSTANT | JANET_SLOT_REF)) {
        /////////    ///////////////////
        janetc_loadconst(c, src.constant, dest);
        ////////////////    ////////////
    // ...
```

Since the slot (`src`) was created via `janetc_cslot`, `src.flags` will have `JANET_SLOT_CONSTANT` set.  Therefore, the first `if` will apply.

Thus, `janetc_loadconst` will be called with our wrapped value `11` (referred to via `src.constant`).

---

```c
/* Load a constant into a local register */
static void janetc_loadconst(JanetCompiler *c, Janet k, int32_t reg) {
    switch (janet_type(k)) {
            /////////////
        case JANET_NIL:
        // ...
        case JANET_BOOLEAN:
        // ...
        case JANET_NUMBER: {
            double dval = janet_unwrap_number(k);
            if (dval < INT16_MIN || dval > INT16_MAX)
                goto do_constant;
            int32_t i = (int32_t) dval;
            if (dval != i)
                goto do_constant;
            uint32_t iu = (uint32_t)i;
            janetc_emit(c,
            ///////////
                        (iu << 16) |
                        (reg << 8) |
                        JOP_LOAD_INTEGER);
                        ////////////////
            break;
    // ...
```

Since `k` (previously `src.constant`) is our wrapped value `11`, the `JANET_NUMBER` case branch is taken.  Since the wrapped value is `11`,  the two `if` checks fail and this leads to a call of `janetc_emit` with `JOP_LOAD_INTEGER`.

---

```c
/* Emit a raw instruction with source mapping. */
void janetc_emit(JanetCompiler *c, uint32_t instr) {
    janet_v_push(c->buffer, instr);
    //////////// /////////  /////
    janet_v_push(c->mapbuffer, c->current_mapping);
}
```

`janetc_emit` takes a single bytecode instruction (`instr`) and using `janet_v_push`, appends it to `c->buffer` (which is where the `JanetCompiler` value, `c`, accumulates emitted bytecode).

Returning from `janetc_emit` puts us back in `janetc_loadconst`.  Then another return brings us to `janetc_movenear` which also returns.  Thus we end up in `janetc_regfar`.

---

```c
/* Convert a slot to a two byte register */
static int32_t janetc_regfar(JanetCompiler *c, JanetSlot s, JanetcRegisterTemp tag) {
    // ...
    int32_t reg;
    int32_t nearreg = janetc_regalloc_temp(&c->scope->ra, tag);
    janetc_movenear(c, nearreg, s);
    ///////////////
    if (nearreg >= 0xF0) {
      // ...
    } else {
        reg = nearreg;
        ///   ///////
        janetc_regalloc_freetemp(&c->scope->ra, nearreg, tag);
        ////////////////////////
        janetc_regalloc_touch(&c->scope->ra, reg);
        /////////////////////
    }
    return reg;
    ////// ///
}

```

Having returned from `janetc_movenear`, `nearreg`'s value is copied to `reg` and `nearreg` itself is "freed".  Then, to hold on to `reg` (or stated differently, to keep `reg` valid), `janetc_regalloc_touch` is called for `reg`.

Returning `reg` from `janetc_regfar`, we are back in `janetc_emit_s`.

---

```c
int32_t janetc_emit_s(JanetCompiler *c, uint8_t op, JanetSlot s, int wr) {
    int32_t reg = janetc_regfar(c, s, JANETC_REGTEMP_0);
            ///   /////////////
    int32_t label = janet_v_count(c->buffer);
    janetc_emit(c, op | (reg << 8));
    ///////////
    // ...
    return label;
    //////
}
```

With `reg` in hand, `janetc_emit` is called.

Note that `op` is `JANET_RETURN`, as this was passed to this function invocation earlier.

Returning brings us back to `janetc_return`, `janetc_value`, and then up to `janet_compile_lint`.

---

```c
/* Compile a form. */
JanetCompileResult janet_compile_lint(Janet source,
                                      JanetTable *env, const uint8_t *where, JanetArray *lints) {
    // ...
    /* Compile the value */
    janetc_value(fopts, source);
    ////////////        //////

    if (c.result.status == JANET_COMPILE_OK) {
        JanetFuncDef *def = janetc_pop_funcdef(&c);
                            //////////////////
    // ...
```

Back from `janetc_value`, with a successful compilation, a call is made to `janetc_pop_funcdef`.

---

```c
/* Compile a funcdef */
/* Once the various other settings of the FuncDef have been tweaked,
 * call janet_def_addflags to set the proper flags for the funcdef */
JanetFuncDef *janetc_pop_funcdef(JanetCompiler *c) {
    JanetScope *scope = c->scope;
    JanetFuncDef *def = janet_funcdef_alloc();
    // ...
    /* Copy bytecode (only last chunk) */
    def->bytecode_length = janet_v_count(c->buffer) - scope->bytecode_start;
    ////////////////////   ////////////////////////   /////////////////////
    if (def->bytecode_length) {
        size_t s = sizeof(int32_t) * (size_t) def->bytecode_length;
        def->bytecode = janet_malloc(s);
        if (NULL == def->bytecode) {
            JANET_OUT_OF_MEMORY;
        }
        safe_memcpy(def->bytecode, c->buffer + scope->bytecode_start, s);
        /////////// /////////////  /////////////////////////////////
        // ...
```

`janetc_pop_funcdef` is responsible for copying a portion of `c->buffer `to `def->bytecode`.

Recall that `c->buffer` is where `janetc_emit` copied bytecode instructions into.

---

...and that is how something can go to 11.

---
