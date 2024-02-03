---
title: "Clarifying rules for brace elision in aggregate initialization"
document: P3106R0
date: today
audience: CWG
author:
    - name: James Touton
      email: <bekenn@gmail.com>
---

## Introduction

@CWG2149 points out an inconsistency in the wording
with respect to array lengths inferred from braced initializer lists
in the presence of brace elision.
The wording has changed since the issue was raised,
but the essence of the issue remains.
The standard first states
that the length of an array of unknown bound
is the same as the number of elements
in the initializer list,
but then describes the rules for brace elision,
in which some elements are used
to initialize subobjects of aggregate members,
such that there is not a one-to-one mapping
of initializer list elements to array elements.
Similarly, aggregate initialization from a braced initializer list
is supposed to explicitly initialize a number of aggregate elements
equal to the length of the list,
which is not possible in the presence of brace elision.

This paper aims to resolve the core issue
by clarifying the rules for brace elision
to match user expectations
and the behavior of existing compilers.
The intent is to better describe the existing design
without introducing any evolutionary changes.

## Wording

All changes are presented relative to @N4971.

&sect;[dcl.init.aggr]{.sref}:

Modify the second list item of paragraph 3:

::: std
>
> - [3.2]{.pnum} If the initializer list is a brace-enclosed *initializer-list*,
>   the explicitly initialized elements of the aggregate
>   are [the first `n` elements of the aggregate,
>   where `n` is the number of elements in the initializer list]{.rm}
>   [those for which an element of the initializer list appertains
>   to the aggregate element or to a subobject thereof (see below)]{.add}.
:::

Modify the second list item of paragraph 4:

::: std
>
> - [4.2]{.pnum} Otherwise, [if the initializer list is
>   a brace-enclosed *designated-initializer-list*,]{.add}
>   the element is [copy-initialized
>   from the corresponding *initializer-clause*
>   or is]{.rm} initialized with the *brace-or-equal-initializer*
>   of the corresponding *designated-initializer-clause*.
>   If that initializer is of the form
>   [*assignment-expression* or]{.rm}
>   `=@&nbsp;@`*assignment-expression*
>   and a narrowing conversion ([dcl.init.list]{.sref}) is required
>   to convert the expression, the program is ill-formed.
>
>   ::: note
>   [If the initialization is by *designated-initializer-clause*,
>   its]{.rm} [The]{.add} form [of the initializer]{.add}
>   determines whether copy-initialization or direct-initialization
>   is performed.
>   :::
>
>   ::: {.note .del}
>   If an initializer is itself an initializer list, *[...]*
>   :::
>
>   ::: {.example .del }
>   *[...]*
>   :::
:::

Add a new list item to paragraph 4:

::: {.std .ins}
>
> - [4.3]{.pnum} Otherwise, the initializer list is a brace-enclosed *initializer-list*.
>   If an *initializer-clause* appertains to the aggregate element,
>   then the aggregate element is copy-initialized from the *initializer-clause*.
>   Otherwise, the aggregate element is copy-initialized from a brace-enclosed
>   *initializer-list* consisting of all of the *initializer-clause*s that
>   appertain to subobjects of the aggregate member, in the order that they appear.
>
>   ::: note
>   If an initializer is itself an initializer list, *[...]*
>   :::
>
>   ::: example
>   *[...]*
>   :::
:::

The note and example that are struck from 4.2
should appear verbatim in 4.3.

Modify paragraph 6:

::: std
> [6]{.pnum}
>
> ::: example
>
> ```c++
> struct S { int a; const char* b; int c; int d = b[a]; };
> S ss = { 1, "asdf" };
> ```
>
> initializes `ss.a` with 1,
> `ss.b` with `"asdf"`,
> `ss.c` with the value of an expression of the form `int{}`
> (that is, `0`), and `ss.d` with the value of `ss.b[ss.a]`
> (that is, `'s'`), and in
>
> ::: del
>
> ```c++
> struct X { int i, j, k = 42; };
> X a[] = { 1, 2, 3, 4, 5, 6 };
> X b[2] = { { 1, 2, 3 }, { 4, 5, 6 } };
> ```
>
> `a` and `b` have the same value
> :::
>
> ```c++
> struct A {
>   string a;
>   int b = 42;
>   int c = -1;
> };
> ```
>
> `A{.c=21}` has the following steps:
>
> - [6.1]{.pnum} Initialize `a` with `{}`
> - [6.2]{.pnum} Initialize `b` with `= 42`
> - [6.3]{.pnum} Initialize `c` with `= 21`
>
> :::
:::

Modify paragraph 10:

::: std
> [10]{.pnum} [An]{.rm} [The number of elements ([dcl.array]{.sref}) in an]{.add} array of unknown bound
> initialized with a
> brace-enclosed *initializer-list*
> [containing `n` *initializer-clause*s]{.rm}
> is defined as [having `n` elements ([dcl.array]{.sref})]{.rm}
> [equal to the number of explicitly initialized elements
> of the array]{.add}.
>
> ::: example
>
> ```c++
> int x[] = { 1, 3, 5 };
> ```
>
> declares and initializes `x`
> as a one-dimensional array that has three elements
> since no size was specified and there are three initializers.
> :::
>
> ::: {.example .ins}
> In
>
> ```c++
> struct X { int i, j, k; };
> X a[] = { 1, 2, 3, 4, 5, 6 };
> X b[2] = { { 1, 2, 3 }, { 4, 5, 6 } };
> ```
>
> `a` and `b` have the same value.
> :::
>
> An array of unknown bound shall not be initialized with
> an empty *braced-init-list* `{}`.
> [The syntax provides for empty *braced-init-list*s,
> but nonetheless C++ does not have zero length arrays.]{.footnote}
>
> ::: note
> A default member initializer does not determine the bound for a member
> array of unknown bound.  Since the default member initializer is
> ignored if a suitable *mem-initializer* is present ([class.base.init]{.sref}),
> the default member initializer is not
> considered to initialize the array of unknown bound.
>
> ::: example
>
> ```c++
> struct S {
>   int y[] = { 0 };          // error: non-static data member of incomplete type
> };
> ```
>
> :::
> :::
:::

<!--
::: std
> [12]{.pnum} An *initializer-list*
> is ill-formed if [the number of *initializer-clause*s
> exceeds the number of elements of the aggregate]{.rm}
> [it contains elements that do not appertain to
> elements of the aggregate or subobjects thereof]{.add}.
>
> ::: example
>
> ```c++
> char cv[4] = { 'a', 's', 'd', 'f', 0 };     // error
> ```
>
> is ill-formed.
>
> :::
:::
-->

Remove paragraph 12:

::: {.std .del}
> [12]{.pnum} An *initializer-list*
> is ill-formed if the number of *initializer-clause*s
> exceeds the number of elements of the aggregate.
>
> ::: example
>
> ```c++
> char cv[4] = { 'a', 's', 'd', 'f', 0 };     // error
> ```
>
> is ill-formed.
>
> :::
:::

Remove paragraph 14:

::: {.std .del}
> [14]{.pnum} If an aggregate class `C` contains a subaggregate element
> `e` with no elements,
> the *initializer-clause* for `e` shall not be
> omitted from an *initializer-list* for an object of type
> `C` unless the *initializer-clause*s for all
> elements of `C` following `e` are also omitted.
>
> ::: example
> *[...]*
> :::
:::

The example in the above paragraph is relocated to another paragraph below.

Remove paragraph 16:

::: {.std .del}
> [16]{.pnum} Braces can be elided in an *initializer-list* as follows.
> If the *initializer-list* begins with a left brace,
> then the succeeding comma-separated list of *initializer-clause*s
> initializes the elements of a subaggregate;
> it is erroneous for there to be more *initializer-clause*s
> than elements.
> If, however, the *initializer-list*
> for a subaggregate does not begin with a left brace,
> then only enough *initializer-clause*s
> from the list are taken to initialize the elements of the subaggregate;
> any remaining *initializer-clause*s
> are left to initialize the next element of the aggregate
> of which the current subaggregate is an element.
>
> ::: example
> *[...]*
> :::
:::

The example in the above paragraph is relocated to another paragraph below.

Insert a new paragraph in place of removed paragraph 16:

::: {.std .ins}
> [14]{.pnum} Each *initializer-clause* in a brace-enclosed *initializer-list*
> is said to *appertain* to an element or subobject thereof
> of the aggregate being initialized.
> Beginning with the first aggregate member and the first *initializer-clause*,
> each *initializer-clause* appertains to the corresponding aggregate member
> if
>
> - [14.1]{.pnum} the aggregate member is not an aggregate, or
> - [14.2]{.pnum} the *initializer-clause* begins with a left brace, or
> - [14.3]{.pnum} the *initializer-clause* is an expression
>   whose type is convertible to the cv-unqualified type
>   of the aggregate member, or
> - [14.4]{.pnum} the aggregate member is an aggregate
>   that itself has no aggregate members.
>
> Otherwise, the members of the subaggregate are considered
> in place of the subaggregate.
>
> ::: note
> These rules are applied recursively to the aggregate's subaggregates,
> and to subaggregates of the subaggregates, and so on.
>
> ::: example
>
> In
>
> ```c++
> struct S1 { int a, b; };
> struct S2 { S1 s, t; };
>
> S2 x[2] = { 1, 2, 3, 4, 5, 6, 7, 8 };
> S2 y[2] = {
>   {
>     { 1, 2 },
>     { 3, 4 }
>   },
>   {
>     { 5, 6 },
>     { 7, 8 }
>   }
> };
> ```
>
> `x` and `y` have the same value.
> :::
> :::
>
> This process continues until all *initializer-clause*s have been exhausted.
>
> If any *initializer-clause*s remain
> after all members of the top-level aggregate
> have been exhausted,
> the program is ill-formed.
>
> ::: example
>
> ```c++
> char cv[4] = { 'a', 's', 'd', 'f', 0 };     // error
> ```
>
> is ill-formed.
>
> :::
:::

Modify paragraph 17 (now 15):

::: std
> [15]{.pnum} All implicit type conversions ([conv]{.sref}) are considered when
> initializing [the]{.rm} [an]{.add} element with an *assignment-expression*.
> [If the *assignment-expression* can initialize an element, the element is initialized.
> Otherwise, if the element is itself a subaggregate,
> brace elision is assumed and the *assignment-expression*
> is considered for the initialization of the first element of the subaggregate.]{.rm}
>
> ::: {.note .del}
> As specified above, brace elision cannot apply to
> subaggregates with no elements; an *initializer-clause* for the entire subobject is
> required.
> :::
>
> ::: {.example .del}
>
> ```c++
> struct A {
>   int i;
>   operator int();
> };
> struct B {
>   A a1, a2;
>   int z;
> };
> A a;
> B b = { 4, a, a };
> ```
>
> Braces are elided around the *initializer-clause*
> for `b.a1.i`.
> `b.a1.i` is initialized with 4,
> `b.a2` is initialized with `a`,
> `b.z` is initialized with whatever `a.operator int()` returns.
> :::
:::

Insert the remaining new paragraphs
(containing the examples relocated from prior paragraphs)
after the above paragraph:

::: {.std .ins}
> [16]{.pnum}
>
> ::: example
>
> ```c++
> float y[4][3] = {
>   { 1, 3, 5 },
>   { 2, 4, 6 },
>   { 3, 5, 7 },
> };
> ```
>
> is a completely-braced initialization:
> 1, 3, and 5 initialize the first row of the array `y[0]`,
> namely `y[0][0]`, `y[0][1]`, and `y[0][2]`.
> Likewise the next two lines initialize `y[1]` and `y[2]`.
> The initializer ends early and therefore `y[3]`'s
> elements are initialized as if explicitly initialized with an
> expression of the form `float()`,
> that is, are initialized with `0.0`.
> In the following example, braces in the *initializer-list* are elided;
> however the *initializer-list*
> has the same effect as the completely-braced *initializer-list*
> of the above example,
>
> ```c++
> float y[4][3] = {
>   1, 3, 5, 2, 4, 6, 3, 5, 7
> };
> ```
>
> The initializer for `y`
> begins with a left brace, but the one for `y[0]` does not,
> therefore three elements from the list are used.
> Likewise the next three are taken successively for `y[1]` and `y[2]`.
>
> :::
:::

::: {.std .ins}
> [17]{.pnum}
>
> ::: note
> The initializer for an empty subaggregate is required
> if any initializers are provided for subsequent elements.
>
> ::: example
>
> ```c++
> struct S { } s;
> struct A {
>   S s1;
>   int i1;
>   S s2;
>   int i2;
>   S s3;
>   int i3;
> } a = {
>   { },              // Required initialization
>   0,
>   s,                // Required initialization
>   0
> };                  // Initialization not required for @`A::s3`@ because @`A::i3`@ is also not initialized
> ```
>
> :::
> :::
:::

::: {.std .ins}
> [18]{.pnum}
>
> ::: example
>
> ```c++
> struct A {
>   int i;
>   operator int();
> };
> struct B {
>   A a1, a2;
>   int z;
> };
> A a;
> B b = { 4, a, a };
> ```
>
> Braces are elided around the *initializer-clause*
> for `b.a1.i`.
> `b.a1.i` is initialized with 4,
> `b.a2` is initialized with `a`,
> `b.z` is initialized with whatever `a.operator int()` returns.
> :::
:::
