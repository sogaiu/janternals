
# Janet's lenprefix PEG Special

---

Some exploration of the lenprefix peg special.

---

```
3AAA2BB1C
```

---

Imagine some data where there is the numeral 3 followed by three
letter A's, then the numeral 2 followed by two letter B's, and
finally, the numeral 1 followed by a single letter C.

So the data has portions where there is a number that specifies the
length of immediately subsequent data followed by such data.

Janet's `lenprefix` peg special can be used to process this type of
data.

---

```janet
(peg/match ~(lenprefix (number :d+) (capture 1))
           "3AAA")
# =>
@["A" "A" "A"]
```

---

Here is a simple example of using `lenprefix`.

`peg/match` is passed two arguments.

The first is a peg expression involving `lenprefix` and the second is
the string that starts with the numeral 3 followed by three letter
A's.

The result is an array with three strings of length one, where each
string is the letter "A".

---

```
(lenprefix n patt)
```

---

`lenprefix` takes two arguments, `n` and `patt`.

`n` should be a peg expression that produces a numeric
capture.

`lenprefix` will apply the peg expression `n` and this should result
in the capture stack ending up with a number at its top.

`lenprefix` will then apply the `patt` peg expression repeatedly as
many times as the aforementioned number at the top of the capture
stack.

Note that before `patt` is applied, captures that got added to the
capture stack (as a result of applying the peg expression `n`) are
removed from the capture stack.

---

```janet
(peg/match ~(any
              (accumulate (lenprefix (number :d+) (capture 1))))
           "3AAA2BB1C")
# =>
@["AAA" "BB" "C"]
```

---

Now consider this example.

Here the `accumulate` peg special is used to concatenate the
individual captures to produce the strings "AAA", "BB", and "C".

The `any` peg special is used to handle the three length-prefixed
segments of data.

---

```janet
(peg/match ~(sequence (number :d+ nil :tag)
                      (capture (lenprefix (backref :tag) 1)))
           "3abc")
# =>
@[3 "abc"]
```

---

Here is another way of arranging for the top-most item of the capture
stack to end up as a number.

In this approach, before the occurrence of `lenprefix`, a capture is
tagged.

Then, the `backref` peg special is used as the first argument to
`lenprefix` using the corresponding tag.

Note that the number used as the length for repetition remains on the
capture stack which may be undesirable in some cases.

However, it's possible to apply the `cmt` or `replace` peg specials to
prevent the number from ultimately ending up on the capture stack.

---

```
`argument`, `constant`
`cmt`, `replace`
`column`, `line`, `position`
`int`, `int-be`, `uint`, `uint-be`
`nth`
```

---

There are also other capturing peg specials that do or can yield
numbers.

These include `argument`, `constant`, `cmt`, `replace`, `column`,
`line`, `position`, `int`, `int-be`, `uint`, `uint-be`, and `nth`.

Note that the `argument` and `constant` peg specials may not be so
useful in the context of `lenprefix` because the associated numbers
need to be known before the call to `peg/match`.

---

## References

* [janet-pegdoc (pdoc) repository](https://github.com/sogaiu/janet-pegdoc/) - https://github.com/sogaiu/janet-pegdoc/
  * [Cheatsheet](https://github.com/sogaiu/janet-pegdoc#cheatsheet) - https://github.com/sogaiu/janet-pegdoc#cheatsheet
* [`lenprefix` implementation in janet's source](https://github.com/janet-lang/janet/blob/73334f34857b0124546ce79b4bda094a4b18a019/src/core/peg.c#L708-L738) - janet/src/core/src/peg.c

---

For some further information, please see these references.

---
