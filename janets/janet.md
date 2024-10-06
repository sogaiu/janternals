---
marp: true
theme: default
class: invert
paginate: true
---

## The Janet Type

---

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

A number of the `janet*` C functions take a value of type `Janet` as a parameter.

In a typical environment, a `Janet` is an 8-byte (64-bit) union.

Boolean values and `nil` are stored in `u64`, numbers in the `number` member, and other values (e.g. `JanetString`s, `JanetArray`s, etc.) in the `pointer` member.

`i64` and `u64` are used for various internal purposes (e.g. converting to and from pointers).

---

```c
#define JANET_NANBOX_TAGBITS     0xFFFF800000000000llu
#define JANET_NANBOX_PAYLOADBITS 0x00007FFFFFFFFFFFllu
```

The "type" (or "tag") of a 64-bit `Janet` value is stored in the upper (17) bits while the "value" (or "payload") is stored in the rest of the (47) bits.

Note that since some of the bits are used for tagging, less than 64 bits are available for expressing values.

Perhaps surprisingly, this can work because not all 64 bits are needed to represent pointers for typical 64-bit architectures nor for non-NaN doubles.

---

```c
#define janet_type(x) \
    (isnan((x).number) \
        ? (JanetType) (((x).u64 >> 47) & 0xF) \
        : JANET_NUMBER)
```

One can determine the type of a `Janet` value via the `janet_type` macro.

A check for `NaN` is done on `x`'s `number` member, and if true, the rightmost 4 bits (0xF == 1111 base 2) of the tag bits are cast to `JanetType`.  Otherwise, the result is `JANET_NUMBER`.

---

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

The `JanetType` enum has 16 (one more than `0xF` == `1111` base 2) possibile values, one for each type of value a `Janet` can wrap.

---

```c
#define janet_unwrap_number(x) ((x).number)
```


To "get at" a `JanetNumber` for a `Janet` value `x`, one can use the `janet_unwrap_number` macro.  The macro simply accesses the `number` member of the union.

---

```c
#define janet_unwrap_string(x) ((JanetString)janet_nanbox_to_pointer(x))
```

Similarly, to "get at" a `JanetString` for a `Janet` value `x`, the `janet_unwrap_string` macro can be used.  It makes use of the `janet_nanbox_to_pointer` function.

---

```c
void *janet_nanbox_to_pointer(Janet x) {
    x.i64 &= JANET_NANBOX_PAYLOADBITS;
    return x.pointer;
}
```

Only the payload bits of `x` are retained (or equivalently, the tag bits are discarded) and then (the modified) `x`'s `pointer` member is returned.

---

```c
#define janet_checktype(x, t) \
    (((t) == JANET_NUMBER) \
        ? janet_nanbox_isnumber(x) \
        : janet_nanbox_checkauxtype((x), (t)))
```

```c
#define janet_nanbox_isnumber(x) \
    (!isnan((x).number) || ((((x).u64 >> 47) & 0xF) == JANET_NUMBER))
```

```c
#define janet_nanbox_checkauxtype(x, type) \
    (((x).u64 & JANET_NANBOX_TAGBITS) == janet_nanbox_tag((type)))
```

```c
#define janet_nanbox_lowtag(type) ((uint64_t)(type) | 0x1FFF0)
#define janet_nanbox_tag(type) (janet_nanbox_lowtag(type) << 47)
```

`janet_checktype` can be used to check if a `Janet` is wrapping a specific `JanetType` (e.g. `JanetString`).

---

```c
#define janet_wrap_string(s) janet_nanbox_wrap_c((s), JANET_STRING)
```

```c
#define janet_nanbox_wrap_c(p, t) \
    janet_nanbox_from_cpointer((p), janet_nanbox_tag(t))
```

```c
Janet janet_nanbox_from_cpointer(const void *p, uint64_t tagmask) {
    Janet ret;
    ret.pointer = (void *)p;
    ret.u64 |= tagmask;
    return ret;
}
```

For the reverse direction of "wrapping", the appropriate callables exist in Janet's API.

For example, the `janet_wrap_string` macro produces a `Janet` from a `JanetString`.

See `janet.h` for more details.


