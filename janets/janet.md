
# The Janet type

---

A brief examination of the `Janet` type and some related functions and macros.

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

---

A number of janet's C functions take a value of type `Janet` as a parameter.

In a typical environment, a `Janet` is an 8-byte (64-bit) union.

A boolean value or a `nil` is stored as a `u64` member, a number as a `number` member, and any other value such as a `JanetString` or `JanetArray` using the `pointer` member.

`i64` and `u64` are used for various internal purposes such as converting to and from pointers.

---

```c
#define JANET_NANBOX_TAGBITS     0xFFFF800000000000llu
#define JANET_NANBOX_PAYLOADBITS 0x00007FFFFFFFFFFFllu
```

---

The upper 17 bits in a 64-bit `Janet` are used to store information about the kind of value it represents such as a `JanetString`.  These bits are referred to as "tag bits".

The remaining lower 47 bits are used for the specifics of the value such as for `true` or `11`.  These bits are referred to as "payload bits".

Note that since some of the 64 bits are used for tagging, less than 64 bits are available for expressing values.

Perhaps surprisingly, this can work because not all 64 bits are needed to represent pointers for typical 64-bit architectures nor for ordinary doubles.

---

```c
#define janet_type(x) \
    (isnan((x).number) \
        ? (JanetType) (((x).u64 >> 47) & 0xF) \
        : JANET_NUMBER)
```

---

The "type" of a `Janet` value can be determined by using the `janet_type` macro.

First, `x`'s `number` member is checked for "not-a-number"-ness.  If the result is true, the rightmost 4 bits of the tag bits are cast to `JanetType`.  Otherwise, the result is `JANET_NUMBER`.

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

---

The `JanetType` enumeration has 16 possible values, one for each type of value a `Janet` can represent or "wrap".

Note that 16 is two raised to the fourth power and recall that the rightmost 4 bits of the tag bits are used to determine what kind of thing a particular `Janet` value represents.

---

```c
#define janet_unwrap_number(x) ((x).number)
```

---

To "get at" or "unwrap" a `JanetNumber` for a `Janet` value `x`, one can use the `janet_unwrap_number` macro.  The macro simply accesses the `number` member of the `Janet` union.

---

```c
#define janet_unwrap_string(x) ((JanetString)janet_nanbox_to_pointer(x))
```

---

Similarly, to "get at" or "unwrap" a `JanetString` for a `Janet` value `x`, the `janet_unwrap_string` macro can be used.  It makes use of the `janet_nanbox_to_pointer` function.

---

```c
void *janet_nanbox_to_pointer(Janet x) {
    x.i64 &= JANET_NANBOX_PAYLOADBITS;
    return x.pointer;
}
```

---

Only the payload bits of `x` are retained (or equivalently, the tag bits are discarded) and then (the modified) `x`'s `pointer` member is returned.

---

```c
#define janet_checktype(x, t) \
    (((t) == JANET_NUMBER) \
        ? janet_nanbox_isnumber(x) \
        : janet_nanbox_checkauxtype((x), (t)))

#define janet_nanbox_isnumber(x) \
    (!isnan((x).number) || ((((x).u64 >> 47) & 0xF) == JANET_NUMBER))

#define janet_nanbox_checkauxtype(x, type) \
    (((x).u64 & JANET_NANBOX_TAGBITS) == janet_nanbox_tag((type)))

#define janet_nanbox_lowtag(type) ((uint64_t)(type) | 0x1FFF0)
#define janet_nanbox_tag(type) (janet_nanbox_lowtag(type) << 47)
```

---

`janet_checktype` can be used to check if a `Janet` is wrapping a specific `JanetType` (for example a `JanetString`).  Note the `& 0xF` for selecting the 4 bits that represent a `JanetType`.

---

```c
#define janet_wrap_string(s) janet_nanbox_wrap_c((s), JANET_STRING)

#define janet_nanbox_wrap_c(p, t) \
    janet_nanbox_from_cpointer((p), janet_nanbox_tag(t))

Janet janet_nanbox_from_cpointer(const void *p, uint64_t tagmask) {
    Janet ret;
    ret.pointer = (void *)p;
    ret.u64 |= tagmask;
    return ret;
}
```

---

For the reverse direction of "wrapping", the appropriate macros and functions exist in Janet's API.

For example, the `janet_wrap_string` macro yields a `Janet` from a `JanetString`.

---

## References

* `janet.h` - wrapping / unwrapping bits, `Janet`, `JanetType`, etc.
* [Pointer magic for efficient dynamic value representations](https://www.npopov.com/2012/02/02/Pointer-magic-for-efficient-dynamic-value-representations.html) - Nikita Popov

---

For some further information, please see these references.

---
