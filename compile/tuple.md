---
marp: true
theme: default
class: invert
paginate: true
---

# Compiling a Tuple to Bytecode

---

Suppose:

```lisp
(def a-tuple '(print "smile!"))

(compile a-tuple)
```

Let's trace how we end up with the corresponding Janet bytecode.

---

A peek at the execution path...

```
cfun_compile
  janet_compile_lint
    janetc_value - tuple
      janetc_value - symbol
        janetc_resolve
      janetc_toslots
        janetc_value - string
          janetc_cslot
        janet_v_push
      janetc_call
        janetc_pushslots
          janetc_emit_s
            janetc_regfar
              janetc_movenear
                janetc_loadconst
                  janetc_emit
                    janet_v_push
            janetc_emit
        janetc_gettarget
        janetc_emit_ss
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

Among other things, `cfun_compile` calls `janet_compile_lint`, passing a janet value that wraps `(print "smile!")` via the parameter `source`.

In this material, the phrase "janet _" (e.g. "janet tuple") will be used though strictly speaking it would be more accurate to say "a _ wrapped as a janet value".

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
    // ...
    /* Compile the value */
    janetc_value(fopts, source);
    ////////////        //////

    if (c.result.status == JANET_COMPILE_OK) {
        JanetFuncDef *def = janetc_pop_funcdef(&c);
                            //////////////////
    // ...
```

After some preparation, `janet_compile_lint` calls `janetc_value`, passing our janet tuple `(print "smile!")` as `source`.

If the compilation is successful, `janet_compile_lint` calls `janetc_pop_funcdef`, but we'll return to this much later.

---

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
    // ...
```

Because `janetc_value` is passed our janet tuple `(print "smile!")`, it recursively calls `janetc_value` with the janet symbol `print` (`tup[0]`).

---

```c
/* Compile a single value */
JanetSlot janetc_value(JanetFopts opts, Janet x) {
    // ...
    if (spec) {
        // ...
    } else {
        switch (janet_type(x)) {
            case JANET_TUPLE: {
            // ...
            case JANET_SYMBOL:
                ret = janetc_resolve(c, janet_unwrap_symbol(x));
                      //////////////
                break;
    // ...
```

Since `print` is a symbol, `janetc_resolve` is called.  However, this path will not be examined.

---

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

Resuming from the `janetc_value` call,  `janetc_call` is called with the return value of `janetc_toslots` (which is passed the rest of the janet tuple `("smile!")` (from `tup + 1` to the end of the janet tuple)).

---

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

`janetc_toslots` returns a pointer to a succession of `JanetSlot`s by accumulating (via `janet_v_push`) individual `JanetSlot` values produced via `janetc_value`.

Each `janetc_value` call is passed a `vals[i]` -- just the janet value `"smile!"` in our case.

---

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
The `janetc_value` call in `janetc_toslots` leads to a call to `janetc_cslot` (because `vals[i]` is the janet string value `"smile!"`).

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

Note that the slot's `flags` field includes `JANET_SLOT_CONSTANT`, the `index` field is `-1`, and the `constant` field is the janet value `"smile!"`.

---

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
        }
        // ...
        } else {
            retslot = janetc_gettarget(opts);
                      ////////////////
        // ...
```

Back at `janetc_call`, it takes `slots` (a `JanetSlot*`) from `janetc_toslots` and calls `janetc_pushslots`.

Eventually, execution will continue after `janetc_pushslots` to reach `janetc_gettarget`, but this will be happening much later...

---

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

`janetc_pushslots` calls `janetc_emit_s` to emit bytecode to push a value referred to by `slots[i]` on the top of the stack.

Note that bytecode being generated doesn't imply it ever gets executed.  The bytecode might eventually be executed or it might not.

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

The primary action of `janetc_emit_s` is to call `janetc_emit`.  In this case, with opcode (`op`) `JOP_PUSH` and the `JanetSlot` (`s`) for our janet value `"smile!"`.

But before calling `janetc_emit`, `janetc_emit_s` calls `janetc_regfar` (passing it the `JanetSlot` value (`s`) for our janet value `"smile!"`).  Time for a bit of a detour...

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

In `janetc_regfar`, since the slot (`s`) was made via `janetc_cslot`, `s.index` is `-1`, so the body of the `if` is skipped.  Thus, `janetc_movenear` will be called.

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

Since the slot (`src`) was created via `janetc_cslot`, `src.flags` will have `JANET_SLOT_CONSTANT` set and thus the first `if` will apply.

Therefore, `janetc_loadconst` will be called with our janet value `"smile!"` (obtained via the `constant` field of our slot `src`).

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
        // ...
        default:
        do_constant: {
                int32_t cindex = janetc_const(c, k);
                janetc_emit(c,
                ///////////
                            (cindex << 16) |
                            (reg << 8) |
                            JOP_LOAD_CONSTANT);
    // ...
```

Since `k` (previously `src.constant`) is our janet value `"smile!"`, the default branch of the switch is taken, which leads to a call of `janetc_emit`.

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

---

```c
int32_t janetc_emit_s(JanetCompiler *c, uint8_t op, JanetSlot s, int wr) {
    int32_t reg = janetc_regfar(c, s, JANETC_REGTEMP_0);
                  /////////////
    // ...
    janetc_emit(c, op | (reg << 8));
    ///////////    //
    // ...
    return label;
    //////
```

We return from our detour (remember?), to end up in `janetc_emit_s`.   After the call to `janetc_regfar`, `janetc_emit` is called to write some bytecode.  Then, after some steps we will not cover, `janetc_emit_s` returns.

As a reminder, `janetc_emit_s` was called by `janetc_pushslots`...

---

```c
int32_t janetc_pushslots(JanetCompiler *c, JanetSlot *slots) {
    // ...
    for (i = 0; i < count;) {
        // ..
        } else if (i + 1 == count) {
            janetc_emit_s(c, JOP_PUSH, slots[i], 0);
            /////////////
     // ...
    return has_splice ? (-1 - min_arity) : min_arity;
    //////
}

```

With the call to `janetc_emit_s` complete, `janetc_pushslots`, also eventually returns to its caller `janetc_call`.

---

```c
/* Compile a call or tailcall instruction */
static JanetSlot janetc_call(JanetFopts opts, JanetSlot *slots, JanetSlot fun) {
    // ...
    if (!specialized) {
        int32_t min_arity = janetc_pushslots(c, slots);
                            ////////////////
    // ...
        }

        if ((opts.flags & JANET_FOPTS_TAIL) &&
                /* Prevent top level tail calls for better errors */
                !(c->scope->flags & JANET_SCOPE_TOP)) {
        // ...
        } else {
            retslot = janetc_gettarget(opts);
            ///////   ////////////////
            janetc_emit_ss(c, JOP_CALL, retslot, fun, 1);
            //////////////    ////////  ///////
        }
    }
    janetc_freeslots(c, slots);
    return retslot;
}
```

Back in `janetc_call`, having returned from `janetc_pushslots`, execution eventually reaches `janetc_gettarget`.  It prepares `retslot`, which is used in a `janetc_emit_ss` call to emit a `JOP_CALL` instruction.

---

...and that my liege, is how we know the world to be banana-shaped.

---

## Afterword: `c->buffer` to `def->bytecode`

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
As noted earlier, `janetc_pop_funcdef` is called by `janet_compile_lint` after it calls `janetc_value`.  `janetc_pop_funcdef` is responsible for copying a portion of `c->buffer `to `def->bytecode`.

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

A check for `NaN` is done on `x`, and if true, `x`'s 64 bits (interpreted as `u64`) are right-shifted by 47 and then the right-most 4 bits (`0xF` == `1111` base 2) are cast to `JanetType`.  Otherwise, `x` is treated as a `JANET_NUMBER`.

---

The different possibilities for `JanetType` are:

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

As an example, to "get at" a janet number for a `Janet` value `x`, one can use the `janet_unwrap_number` macro:

```c
#define janet_unwrap_number(x) ((x).number)
```

---


Similarly, to "get at" a janet string for a `Janet` value `x`, one can use the `janet_unwrap_string` macro:

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

Here, only the payload bits of `x` are retained (or equivalently, the tag bits are discarded) and then `x`'s `pointer` field is returned.

---

As might be expected, for the reverse direction of "wrapping", the appropriate callables exist in Janet's API.

For example:

* `janet_nanbox_wrap_c`
* `janet_nanbox_from_cpointer`
* `janet_wrap_string`

See `janet.h` for more details.

---

## Additional Topics

---

Did not cover:

```
* JanetCompileResult
* JanetCompiler
* JanetFopts
* janetc_resolve
* janetc_freeslot
* JanetFuncDef
* JanetRegisterTemp
* janetc_regalloc_temp
* janetc_const
* janet_v_count
* janet_v_push
* janet_malloc
* JanetScope
* janetc_gettarget
* janetc_emit_ss
```
