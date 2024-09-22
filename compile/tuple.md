---
marp: true
theme: default
class: invert
paginate: true
---

# Compiling `(print "smile!")` to Bytecode

---

Starting with:

```janet
(compile '(print "smile!"))
```

let's trace how we end up with the corresponding Janet bytecode.

---

Much will be skipped, but hope to cover many essentials...

```
* cfun_compile
  * janet_compile_lint
    * janetc_value
      * janetc_value
      * janetc_toslots
        * janetc_value
          * janetc_cslot
        * janet_v_push
      * janetc_call
        * janetc_pushslots
          * janetc_emit_s
            * janetc_regfar
              * janetc_movenear
                * janetc_loadconst
                  * janetc_emit
                    * janet_v_push
            * janetc_emit
    * janetc_pop_funcdef
```

---

The Janet function `compile` is implemented in C by `cfun_compile` in
`compile.c`:

```c
/* C Function for compiling */
JANET_CORE_FN(cfun_compile,
              "(compile ast &opt env source lints)",
              "Compiles an Abstract Syntax Tree (ast) into a function. "
              "Pair the compile function with parsing functionality to implement "
              "eval. Returns a new function and does not modify ast. Returns an error "
              "struct with keys :line, :column, and :error if compilation fails. "
              "If a `lints` array is given, linting messages will be appended to the array. "
              "Each message will be a tuple of the form `(level line col message)`.") {
    // ...
    JanetCompileResult res = janet_compile_lint(argv[0], env, source, lints);
                             //////////////////               //////
    // ...
```

Among other things, `cfun_compile` calls `janet_compile_lint`, passing the janet tuple `(print "smile!")` [1] via the parameter `source`.

[1] In this material, the phrase "janet _" (e.g. "janet tuple") will be used though strictly speaking it would be more accurate to say "a _ wrapped as a janet value".

---

After some preparation, `janet_compile_lint` calls `janetc_value`, passing our janet tuple `(print "smile!")` as `source`:

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
    // ...
    /* Compile the value */
    janetc_value(fopts, source);
    ////////////        //////

    if (c.result.status == JANET_COMPILE_OK) {
        JanetFuncDef *def = janetc_pop_funcdef(&c);
                            //////////////////
    // ...
```

If the compilation is successful, `janet_compile_lint` calls `janetc_pop_funcdef`, but we'll return to this later.

---

Because `janetc_value` is passed our janet tuple `(print "smile!")`, it recursively calls `janetc_value` with the janet symbol `print` (`tup[0]`):

```c
/* Compile a single value */
JanetSlot janetc_value(JanetFopts opts, Janet x) {
    JanetSlot ret;
    JanetCompiler *c = opts.compiler;
    // ...
            case JANET_TUPLE: {
                JanetFopts subopts = janetc_fopts_default(c);
                const Janet *tup = janet_unwrap_tuple(x);
                // ,,,
                } else {
                    JanetSlot head = janetc_value(subopts, tup[0]);
                                     ////////////          //////
                    subopts.flags = JANET_FUNCTION | JANET_CFUNCTION;
                    ret = janetc_call(opts, janetc_toslots(c, tup + 1, janet_tuple_length(tup) - 1), head);
                          ///////////       //////////////    ///////
                    janetc_freeslot(c, head);
                }
    // ...
```

Subsequently, `janetc_call` is called with the return value of `janetc_toslots` (which is passed the rest of the janet tuple `("smile!")` (from `tup + 1` to the end of the janet tuple)).

---

`janetc_toslots` returns a pointer to a succession of `JanetSlot`s by
accumulating (via `janet_v_push`) individual `JanetSlot` values
produced via `janetc_value`:

```c
/* Get a bunch of slots for function arguments */
JanetSlot *janetc_toslots(JanetCompiler *c, const Janet *vals, int32_t len) {
    int32_t i;
    JanetSlot *ret = NULL;
    JanetFopts subopts = janetc_fopts_default(c);
    // ...
    for (i = 0; i < len; i++) {
        janet_v_push(ret, janetc_value(subopts, vals[i]));
        ////////////      ////////////          ///////
    }
    return ret;
}
```

Each `janetc_value` call is passed a `vals[i]` -- just the janet value `"smile!"` in our case.

---

The `janetc_value` call in `janetc_toslots` leads to a call to `janetc_cslot` (because `vals[i]` is the janet string value `"smile!"`):

```c
/* Compile a single value */
JanetSlot janetc_value(JanetFopts opts, Janet x) {
    // ...
    } else {
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
                break;
    // ...
```

---

`janetc_cslot` produces a `JanetSlot` for a constant value:

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

Note that the slot's `flags` field includes `JANET_SLOT_CONSTANT`, the
`index` field is `-1`, and the `constant` field is the janet value `"smile!"`.

---

Returning to `janetc_call`, it takes the `JanetSlot*` (here named
`slots`) produced by `janetc_toslots` and calls `janetc_pushslots`:

```c
/* Compile a call or tailcall instruction */
static JanetSlot janetc_call(JanetFopts opts, JanetSlot *slots, JanetSlot fun) {
    JanetSlot retslot;
    JanetCompiler *c = opts.compiler;
    int specialized = 0;
    // ...
    if (!specialized) {
        int32_t min_arity = janetc_pushslots(c, slots);
                            ////////////////    /////
    // ...
```

---

`janetc_pushslots` calls `janetc_emit_s` to emit bytecode to push
a value referred to by `slots[i]` on the top of the stack:

```c
/* Push slots loaded via janetc_toslots. Return the minimum number of slots pushed,
 * or -1 - min_arity if there is a splice. (if there is no splice, min_arity is also
 * the maximum possible arity). */
int32_t janetc_pushslots(JanetCompiler *c, JanetSlot *slots) {
    int32_t i;
    int32_t count = janet_v_count(slots);
    // ...
    for (i = 0; i < count;) {
        // ...
        } else if (i + 1 == count) {
            janetc_emit_s(c, JOP_PUSH, slots[i], 0);
            /////////////    ////////  ////////
        // ...
}
```

Note that bytecode being generated doesn't imply it ever gets
executed.  The bytecode might eventually be executed or it might not.

---

The primary action of `janetc_emit_s` (`_s` means bytecode with one argument) is to call `janetc_emit`.  In this case, with opcode (`op`) `JOP_PUSH` and the `JanetSlot` (`s`) for our janet value `"smile!"`:

```c
int32_t janetc_emit_s(JanetCompiler *c, uint8_t op, JanetSlot s, int wr) {
                   //
    int32_t reg = janetc_regfar(c, s, JANETC_REGTEMP_0);
                  /////////////
    // ...
    janetc_emit(c, op | (reg << 8));
    ///////////    //
    // ...
```

...but before calling `janetc_emit`, `janetc_emit_s` calls `janetc_regfar` (passing it the `JanetSlot` value (`s`) for our janet value `"smile!"`).  Time for a bit of a detour...

---

In `janetc_regfar`, since the slot (`s`) was made via `janetc_cslot`,
`s.index` is `-1`, so the body of the `if` is skipped.  Thus, `janetc_movenear` will be called:

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
}
```

---

Since the slot (`src`) was createded via `janetc_cslot`, `src.flags` will have `JANET_SLOT_CONSTANT` set and thus the first `if` will apply:

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
}
```

Therefore, `janetc_loadconst` will be called with our janet value `"smile!"`  (obtained via the `constant` field of our slot `src`).

---

Since `k` (previously `src.constant`) is our janet value `"smile!"`, the default branch of the switch is taken, which leads to a call of `janetc_emit`:

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
        // ...
        default:
        do_constant: {
                int32_t cindex = janetc_const(c, k);
                janetc_emit(c,
                ///////////
                            (cindex << 16) |
                            (reg << 8) |
                            JOP_LOAD_CONSTANT);
                break;
            }
    }
}
```

---

`janetc_emit` takes a single bytecode instruction (`instr`) and
using `janet_v_push`, appends it to `c->buffer` (which is where the
`JanetCompiler` value, `c`, accumulates emitted bytecode).

```c
/* Emit a raw instruction with source mapping. */
void janetc_emit(JanetCompiler *c, uint32_t instr) {
    janet_v_push(c->buffer, instr);
    //////////// /////////  /////
    janet_v_push(c->mapbuffer, c->current_mapping);
}
```

---

Returning from out detour (remember?), all the way back to `janetc_emit_s`, after the call to `janetc_regfar`, `janetc_emit` is called again:

```c
int32_t janetc_emit_s(JanetCompiler *c, uint8_t op, JanetSlot s, int wr) {
    int32_t reg = janetc_regfar(c, s, JANETC_REGTEMP_0);
    // ...
    janetc_emit(c, op | (reg << 8));
    ///////////    //
    // ...
```

...and that my liege, is how we know the world to be banana-shaped.

---

## Afterword: `c->buffer` to `def->bytecode`

---

As noted earlier, `janetc_pop_funcdef` is called by `janet_compile_lint` after
it calls `janetc_value`.  `janetc_pop_funcdef` is responsible for copying
a portion of `c->buffer `to `def->bytecode`:

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

---

## Unfinished Business: janet tuple, janet string, etc.

---

A number of the `janetc_*` functions take as a parameter, a value of type `Janet`.  In the author's primary environment, this is defined in `janet.h` as:

```c
/* Recursive type (Janet) */
#ifdef JANET_NANBOX_64
typedef union Janet Janet;
union Janet {
    uint64_t u64;
    int64_t i64;
    double number;
    void *pointer;
};
```

A `Janet` is a union that is capable of representing each of the janet values (e.g. string, array, table, etc.) and locally it is 8 bytes (or 64 bits) [1].

[1] Numbers are stored directly (`u64`, `i64`, and `number` fields), but for other values (e.g. strings, arrays, etc.) a pointer (`pointer` field) is stored.

---

The "type" (or "tag") of a `Janet` value is stored in the upper (17) bits while the "value" (or "payload") is stored in the rest of the (47) bits:

```c
#define JANET_NANBOX_TAGBITS     0xFFFF800000000000llu
#define JANET_NANBOX_PAYLOADBITS 0x00007FFFFFFFFFFFllu
```

---

One can determine the type of a `Janet` value via `janet_type`:

```c
#define janet_type(x) \
    (isnan((x).number) \
        ? (JanetType) (((x).u64 >> 47) & 0xF) \
        : JANET_NUMBER)
```

---

All of the janet "types" are:

```c
/* Basic types for all Janet Values */
typedef enum JanetType {
    JANET_NUMBER,
    JANET_NIL,
    JANET_BOOLEAN,
    JANET_FIBER,
    JANET_STRING,
    JANET_SYMBOL,
    JANET_KEYWORD,
    JANET_ARRAY,
    JANET_TUPLE,
    JANET_TABLE,
    JANET_STRUCT,
    JANET_BUFFER,
    JANET_FUNCTION,
    JANET_CFUNCTION,
    JANET_ABSTRACT,
    JANET_POINTER
} JanetType;
```

---

As an example, to "get at" a janet number, one can use the `janet_unwrap_number` macro:

```c
#define janet_unwrap_number(x) ((x).number)
```

---


To "get at" a janet string, one can use the `janet_unwrap_string` macro:

```c
#define janet_unwrap_string(x) ((JanetString)janet_nanbox_to_pointer(x))
```

which makes use of `janet_nanbox_to_pointer`:

```c
void *janet_nanbox_to_pointer(Janet x) {
    x.i64 &= JANET_NANBOX_PAYLOADBITS;
    return x.pointer;
}
```

---

For the reverse direction of "wrapping", the appropriate callables exist in Janet's API.  For example:

* `janet_nanbox_wrap_c`
* `janet_nanbox_from_cpointer`
* `janet_wrap_string`

See `janet.h` for more details.

---

## Additional Topics

---

Did not cover:

* JanetCompileRest
* JanetCompiler
* JanetFopts
* janetc_freeslot
* JanetFuncDef
* JanetRegisterTemp
* janetc_regalloc_temp
* janetc_const
* janet_v_count
* janet_v_push
* janet_malloc
* JanetScope
