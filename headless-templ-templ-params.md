---
title: "Headless Template Template Parameters"
document: P3158R0
date: today
audience: Evolution
author:
    - name: James Touton
      email: <bekenn@gmail.com>
---

## Introduction

This paper introduces a new flavor of template template parameter
that matches any template template argument,
herein referred to as *headless* template template parameters
because they are declared without a *template-head*.
This feature is intended to be used
with universal template parameters (@P1985R3, @P2989R0),
but may be adopted separately.
This feature obsoletes a form of template argument matching
that allows a template template parameter
with a template parameter pack
to match a template template argument
without a template parameter pack
(violating the contract
implied by the form of the template template parameter).
This paper also introduces constrained template template parameter declarations,
as constraints are expected to be used frequently
in combination with headless template template parameters.

### Universal Template Parameter Syntax

The syntax for the declaration
of a universal template parameter
is as yet undecided.
@P1985R3 and @P2989R0 offer several possibilities,
each of which has drawbacks
associated with
introducing a new keyword,
repurposing an old keyword,
designating an identifier as a contextual keyword,
or some combination of the above.
The examples in those papers
are not consistent in their choice of syntax;
for internal consistency,
and to avoid the issues mentioned above,
this paper will use `?`
to declare a universal template parameter:

```c++
// Example from P2989R0
template <universal template>
constexpr bool is_variable = false;
template <auto a>
constexpr bool is_variable<a> = true;

// Same example, rewritten with @[?]{.tcode}@
template <?>
constexpr bool is_variable = false;
template <auto a>
constexpr bool is_variable<a> = true;
```

### Universal Template Parameter Semantics

Taken together,
@P1985R3 and @P2989R0 present several different models
of universal template parameter semantics.
For the purposes of this paper,
we'll assume the semantics presented in @P2989R0;
that is:

- A universal template parameter binds to any template argument of any kind.
- A template that uses universal parameters
  may have partial specializations
  that refine the kind of a universal parameter.
- A universal template parameter may be used
  as a template argument for a template parameter of any kind;
  if, during instantiation, the substituted value
  is of a kind incompatible with the template parameter,
  the program is ill-formed.
- A universal template parameter may not be used
  in any other context.

### Universal Template Parameter Topics Not Covered

@P1985R3 discusses a number of additional hypothetical language features
that would have interactions with universal template parameters
where they to be adopted.
As these are not currently part of C++,
this paper will not discuss them.
The features are:

- Concept template parameters
- Variable template parameters
- Universal aliases

## Motivation

The primary motivation for this feature
is to make universal template parameters useful
without backsliding on one of the key principles of @P0522R0,
which is that the form of a template template parameter
should be indicative of its proper use within its scope.
A secondary motivation is to provide an alternative
to an existing rule that violates this principle.
These two issues are closely related.

### Template Template Argument Matching

@N2555 introduced a change to template template argument matching
that allows a template template parameter
with a template parameter pack
to match a template template argument
without a template parameter pack.
This was done to grant the ability
to write a metafunction
that could accept a template
and an arbitrary number of template arguments,
and then apply the arguments to the template:

```c++
template <template <class...> class F, class... Args>
using apply = F<Args...>;

template <class T1>           struct A;
template <class T1, class T2> struct B;
template <class... Ts>        struct C;

template <class T> struct X { };

X<apply<A, int>> xa;                              // OK
X<apply<B, int, float>> xb;                       // OK
X<apply<C, int, float, int, short, unsigned>> xc; // OK
```

This addressed the immediate need,
but it did so in a way that
undermines the contract
implied by the *template-head*
of the template template parameter.
In the above example,
it would be reasonable for the author of `apply`
to expect that `F` is a template
that can accept any number of type arguments,
as is the case for `C`.
However, `A` can only accept one argument,
and `B` can only accept two arguments.
The contract implied by the declaration of `F` is violated,
but the parameter is allowed to bind anyway.
If a metafunction actually does require
that the template accept any number of arguments,
then there is no longer any space
in the existing syntax
to express that constraint.
We've taken syntax that ought to mean one thing
and assigned an entirely different meaning to it.

@N2555 was a reasonable compromise for its time.
It addressed a very real need
at the cost of a small syntactic carve-out
in a feature that nobody had seen before (variadic templates).
Today, variadic templates are everywhere,
template constraints permit us
to make fine-grained decisions
based on the content of a template declaration,
and with reflection on the horizon,
the ability to accurately express intent
is more important than ever.
If universal template parameters take an approach similar to @N2555,
the damage will be much worse.

### Universal Template Parameters

In the example above,
`apply` can only operate on templates
taking some number of type arguments.
If you wish to `apply` non-type or template arguments
to a template that can accept them,
you're out of luck.
There are two reasons for this.
The first is that
`F` is specified in such a way
that it cannot bind to a template
that can accept template or non-type arguments.
The second is that `apply` itself
is unable to accept template or non-type arguments in `Args`.
Universal template parameters are an essential feature
well-suited to solving *one* of those problems,
but @P1985R3 attempts to solve both of those issues
with the same feature
by abusing template template parameter matching
in much the same way that @N2555 did:

```c++
template <template <?...> typename F, ?... Args>
using apply = F<Args...>; // easy peasy!

// ok, r1 is std::array<int, 3>
using r1 = apply<std::array, int, 3>;
// ok, r2 is std::vector<int, std::pmr::allocator>
using r2 = apply<std::vector, int, std::pmr::allocator>;
```

As with @N2555,
the parameter pack in the *template-head* of `F` is misleading.
This much follows from the rules established by @N2555;
it is not a new problem,
and we can't change the meaning of the parameter pack
without a lengthy deprecation period
(assuming there's any appetite to try; more on this later).
Unlike @N2555, the parameter kind
(the pattern of the parameter pack)
is *also* misleading.

A naïve reading of the declaration of `F`
would interpret it as a template
capable of taking any number of arguments,
*each of any kind*.
However, `F` will bind to *any* template template argument whatsoever,
irrespective of the number or kinds of template parameters
or even any associated constraints on the argument template.

## Design

In the case of `apply`,
what's actually desired
is a form of template template parameter
that does not suggest a capability
that expresses our lack of knowledge
about the bound template template argument.
In each of the prior formulations of `apply`,
the *template-head* of `F` has conveyed misleading information
about the properties of `F`,
when what we really want is a way to say
that we don't know anything about `F`.
So let's get rid of the *template-head* entirely:

```c++
template <template @[`<?...>`]{.rm}@ typename F, ?... Args>
using apply = F<Args...>; // easy peasy!
```

Now, we can actually distinguish between
a template that accepts any arguments whatsoever
and a template that we know nothing about.
We now have the tools we need to fully express intent.

### Constrained template template parameters

Headless template template parameters are essentially
free of *any* constraints,
as they are not constrained by parameter count or kind.
It seems likely that one of the first things
a programmer will want to do
is to add some form of constraint on these parameters.
The most obvious constraint is a check
for the validity of a set of template arguments:

```c++
template <template typename T, ?... Args>
  concept specializable = requires { typename T<Args...>; };
```

To ease the application of constraints,
a concept name denoting a template concept
may be used to declare a headless template template parameter.
This works exactly as it does for type constraints,
except that the concept's prototype parameter
must be a template template parameter
rather than a type parameter.

```c++
template <specializable<int, 5> TT> void f();

// Same as:
//   template <template typename TT> requires specializable<TT, int, 5>
//   void f();
```

## Examples

Following are some examples copied from @P1985R3.
The examples have been updated with diff marks
showing the difference in syntax
with the application of headless template template parameters.
In all cases, the semantics of the example are unchanged.
As the difference is entirely in the absence of a *template-head*,
headless template template parameters result in
simpler syntax that is also more representative of intent.

```c++
// is_specialization_of
template <typename T, template @[`<?...>`]{.rm}@ typename Type>
constexpr bool is_specialization_of_v = false;

template <?... Params, template @[`<?...>`]{.rm}@ typename Type>
constexpr bool is_specialization_of_v<Type<Params...>, Type> = true;

template <typename T, template @[`<?...>`]{.rm}@ typename Type>
concept specialization_of = is_specialization_of_v<T, Type>;
```

```c++
template<typename Type, template @[`<?...>`]{.rm}@ typename Templ>
constexpr bool is_specialization_of_v = (template_of(^Type) == ^Templ);
```

```c++
template <?> constexpr bool is_typename_v             = false;
template <typename T> constexpr bool is_typename_v<T> = true;
template <?> constexpr bool is_value_v                = false;
template <auto V> constexpr bool is_value_v<V>        = true;
template <?> constexpr bool is_template_v             = false;
template <template @[`<?...>`]{.rm}@ typename A>
constexpr bool is_template_v<A>                       = true;

// The associated type for each trait:
template <? X> struct is_typename : std::bool_constant<is_typename_v<X>> {};
template <? X> struct is_value    : std::bool_constant<is_value_v<X>> {};
template <? X> struct is_template : std::bool_constant<is_template_v<X>> {};
```

```c++
template <?> struct box; // impossible to define body

template <auto X>
struct box<X> { static constexpr decltype(X) result = X; };

template <typename X>
struct box<X> { using result = X; };

template <template @[`<?...>`]{.rm}@ typename X>
struct box<X> {
  template <?... Args>
  using result = X<Args...>;
};
```

```c++
template <template @[`<?>`]{.rm}@ typename Map,
          template @[`<?...>`]{.rm}@ typename Reduce,
          ?... Args>
using map_reduce = Reduce<Map<Args>::result...>;
```

```c++
template <int... xs> using sum = box<(0 + ... + xs)>;
template <? X> using boxed_is_typename = box<is_typename_v<X>>;
static_assert(2 == map_reduce<boxed_is_typename, sum, int, 1, long, std::vector>::result);
```

```c++
template<typename T>
struct unwrap
{
  using result = T;
};

template<typename T, T t>
struct unwrap<std::integral_constant<T, t>>
{
  static constexpr T result = t;
};

template <template @[<?...>]{.rm}@ typename T, typename... Params>
using apply_unwrap = T<unwrap<Params>::result...>;

apply_unwrap<std::array, int, std::integral_constant<std::size_t, 5>> arr;
```

```c++
template <template @[<?...>]{.rm}@ typename F,
          ? ... Args1>
struct curry {
  template <?... Args2>
  using func = F<Args1..., Args2...>;
};
```

```c++
template<template@[<?...>]{.rm}@ class C, typename... Ps>
auto make_unique(Ps&&... ps) {
  return unique_ptr<decltype(C(std::forward<Ps>(ps)...))>(new C(std::forward<Ps>(ps)...));
}
```

## Wording

&sect;[expr.prim.lambda.closure/8]{.sref}

::: std
> [8]{.pnum}
>
> ::: note
> The function call operator or operator template can be constrained ([temp.constr.decl]{.sref})
> by a [*type-constraint*]{.rm} [*constraint-specifier*]{.add} ([temp.param]{.sref}),
> a *requires-clause* ([temp.pre]{.sref}),
> or a trailing *requires-clause* ([dcl.decl]{.sref}).
>
> ::: example
>
> ```c++
> template <typename T> concept C1 = /* ... */;
> template <std::size_t N> concept C2 = /* ... */;
> template <typename A, typename B> concept C3 = /* ... */;
>
> auto f = []<typename T1, C1 T2> requires C2<sizeof(T1) + sizeof(T2)>
>          (T1 a1, T1 b1, T2 a2, auto a3, auto a4) requires C3<decltype(a4), T2> {
>   // @[T2]{.tcode}@ is constrained by a @[*type-constraint*]{.rm} [*constraint-specifier*]{.add}@.
>   // @[T1]{.tcode}@ and @[T2]{.tcode}@ are constrained by a @*requires-clause*@, and
>   // @[T2]{.tcode}@ and the type of @[a4]{.tcode}@ are constrained by a trailing *requires-clause*.
> };
> ```
>
> :::
> :::
:::

&sect;[expr.prim.req.compound]{.sref}

::: std
> |         *compound-requirement*:
> |                 `{` *expression* `}` `noexcept`~*opt*~ *return-type-requirement*~*opt*~ `;`
> |
> |         *return-type-requirement*:
> |                 `->` [*type-constraint*]{.rm} [*constraint-specifier*]{.add}
:::

&sect;[expr.prim.req.compound/1]{.sref}

::: std
> [1]{.pnum}
> A *compound-requirement* asserts properties
> of the *expression* *E*. Substitution
> of template arguments (if any) and verification of
> semantic properties proceed in the following order:
>
> - [1.1]{.pnum}
>   Substitution of template arguments (if any)
>   into the *expression* is performed.
> - [1.2]{.pnum}
>   If the `noexcept` specifier is present,
>   *E* shall not be a potentially-throwing expression ([except.spec]{.sref}).
> - [1.3]{.pnum}
>   If the *return-type-requirement* is present, then:
>
>   - [1.3.1]{.pnum}
>     Substitution of template arguments (if any)
>     into the *return-type-requirement* is performed.
>   - [The *constraint-specifier* shall name a type concept.]{.add}
>   - [1.3.2]{.pnum}
>     The immediately-declared constraint ([temp.param]{.sref})
>     of the [*type-constraint*]{.rm} [*constraint-specifier*]{.add} for `decltype((`*E*`))`
>     shall be satisfied.
>
>   *[...]*
>
> *[...]*
:::

&sect;[dcl.type.simple/2]{.sref}

::: std
> [2]{.pnum}
> The component names of a *simple-type-specifier* are those of its
> *nested-name-specifier*,
> *type-name*,
> *simple-template-id*,
> *template-name*, and/or
> [*type-constraint*]{.rm} [*constraint-specifier*]{.add}
> (if it is a *placeholder-type-specifier*).
> The component name of a *type-name* is the first name in it.
:::

&sect;[dcl.spec.auto.general]{.sref}

::: std
> |         *placeholder-type-specifier*:
> |                 [*type-constraint*~*opt*~]{.rm} [*constraint-specifier*~*opt*~]{.add} `auto`
> |                 [*type-constraint*~*opt*~]{.rm} [*constraint-specifier*~*opt*~]{.add} `decltype` `(` `auto` `)`
:::

&sect;[dcl.spec.auto.general/2]{.sref}

::: std
> [2]{.pnum}
> A *placeholder-type-specifier* of the form
> [*type-constraint*~*opt*~]{.rm} [*constraint-specifier*~*opt*~]{.add} `auto`
> can be used as a *decl-specifier* of
> the *decl-specifier-seq* of
> a *parameter-declaration* of
> a function declaration or *lambda-expression* and,
> if it is not the `auto` *type-specifier*
> introducing a *trailing-return-type* (see below),
> is a *generic parameter type placeholder*
> of the function declaration or *lambda-expression*.
>
> *[...]*
:::

&sect;[dcl.type.auto.deduct/2.1]{.sref}

::: std
>
> - [2.1]{.pnum}
>   For a non-discarded `return` statement that occurs
>   in a function declared with a return type
>   that contains a placeholder type,
>   `T` is the declared return type.
>
>   *[...]*
>
>   If *E* has type `void`,
>   `T` shall be either
>   [*type-constraint*~*opt*~]{.rm} [*constraint-specifier*~*opt*~]{.add} `decltype(auto)` or
>   *cv* [*type-constraint*~*opt*~]{.rm} [*constraint-specifier*~*opt*~]{.add} `auto`.
:::

&sect;[dcl.type.auto.deduct/3]{.sref}

::: std
> [3]{.pnum}
> If the *placeholder-type-specifier* is of the form
> [*type-constraint*~*opt*~]{.rm} [*constraint-specifier*~*opt*~]{.add} `auto`,
> the deduced type
> `T`&prime; replacing `T`
> is determined using the rules for template argument deduction.
> If the initialization is copy-list-initialization,
> a declaration of `std::initializer_list`
> shall precede ([basic.lookup.general]{.sref})
> the *placeholder-type-specifier*.
> Obtain `P` from
> `T` by replacing the occurrences of
> [*type-constraint*~*opt*~]{.rm} [*constraint-specifier*~*opt*~]{.add} `auto` either with
> a new invented type template parameter `U` or,
> if the initialization is copy-list-initialization, with
> `std::initializer_list<U>`. Deduce a value for `U` using the rules
> of template argument deduction from a function call ([temp.deduct.call]{.sref}),
> where `P` is a
> function template parameter type and
> the corresponding argument is *E*.
> If the deduction fails, the declaration is ill-formed.
> Otherwise, `T`&prime; is obtained by
> substituting the deduced `U` into `P`.
>
> *[...]*
:::

&sect;[dcl.type.auto.deduct/4]{.sref}

::: std
> [4]{.pnum}
> If the *placeholder-type-specifier* is of the form
> [*type-constraint*~*opt*~]{.rm} [*constraint-specifier*~*opt*~]{.add} `decltype(auto)`,
> `T` shall be the
> placeholder alone. The type deduced for `T` is
> determined as described in [dcl.type.decltype]{.sref}, as though
> *E* had
> been the operand of the `decltype`.
>
> *[...]*
:::

&sect;[dcl.type.auto.deduct/5]{.sref}

::: std
> [5]{.pnum}
> For a *placeholder-type-specifier*
> with a [*type-constraint*]{.rm} [*constraint-specifier*]{.add},
> [the *constraint-specifier* shall name a type concept ([temp.concept]{.sref}) and]{.add}
> the immediately-declared constraint ([temp.param]{.sref})
> of the [*type-constraint*]{.rm} [*constraint-specifier*]{.add} for the type deduced for the placeholder
> shall be satisfied.
:::

&sect;[dcl.fct/22]{.sref}

::: std
> [22]{.pnum}
> An *abbreviated function template*
> is a function declaration that has
> one or more generic parameter type placeholders ([dcl.spec.auto]{.sref}).
> An abbreviated function template is equivalent to
> a function template ([temp.fct]{.sref})
> whose *template-parameter-list* includes
> one invented type *template-parameter*
> for each generic parameter type placeholder
> of the function declaration, in order of appearance.
> For a *placeholder-type-specifier* of the form `auto`,
> the invented parameter is
> an unconstrained *type-parameter*.
> For a *placeholder-type-specifier* of the form
> [*type-constraint*]{.rm} [*constraint-specifier*]{.add} `auto`,
> the invented parameter is a *type-parameter* with
> that [*type-constraint*]{.rm} [*constraint-specifier*]{.add}.
> The invented type *template-parameter* is
> a template parameter pack
> if the corresponding *parameter-declaration*
> declares a function parameter pack.
> If the placeholder contains `decltype(auto)`,
> the program is ill-formed.
> The adjusted function parameters of an abbreviated function template
> are derived from the *parameter-declaration-clause* by
> replacing each occurrence of a placeholder with
> the name of the corresponding invented *template-parameter*.
>
> *[...]*
:::

&sect;[temp.param/1]{.sref}

::: std
> [1]{.pnum}
> The syntax for *template-parameter*s is:
>
> |         *template-parameter*:
> |                 *type-parameter*
> |                 *parameter-declaration*
> |
> |         *type-parameter*:
> |                 *type-parameter-key* `...`~*opt*~ *identifier*~*opt*~
> |                 *type-parameter-key* *identifier*~*opt*~ `=` *type-id*
> |                 [*type-constraint*]{.rm} [*constraint-specifier*]{.add} `...`~*opt*~ *identifier*~*opt*~
> |                 [*type-constraint*]{.rm} [*constraint-specifier*]{.add} *identifier*~*opt*~ `=` *type-id*
> |                 *template-head* *type-parameter-key* `...`~*opt*~ *identifier*~*opt*~
> |                 *template-head* *type-parameter-key* *identifier*~*opt*~ `=` *id-expression*
> |                 [`template` *type-parameter-key* `...`~*opt*~ *identifier*~*opt*~]{.add}
> |                 [`template` *type-parameter-key* *identifier*~*opt*~ `=` *id-expression*]{.add}
> |
> |         *type-parameter-key*:
> |                 `class`
> |                 `typename`
> |
> |         [*type-constraint*]{.rm} [*constraint-specifier*]{.add}:
> |                 *nested-name-specifier*~*opt*~ *concept-name*
> |                 *nested-name-specifier*~*opt*~ *concept-name* `<` *template-argument-list*~*opt*~ `>`
>
> The component names of a [*type-constraint*]{.rm} [*constraint-specifier*]{.add} are
> its *concept-name* and
> those of its *nested-name-specifier* (if any).
>
> *[...]*
:::

<!--
&sect;[temp.param/2]{.sref}

::: std
> [2]{.pnum}
> There is no semantic difference between `class` and `typename`
> in a *type-parameter-key*.
> `typename` followed by an *unqualified-id*
> names a [template type parameter]{.rm} [*template type parameter*]{.add}.
>
> [\[ *Note:*]{.add}
> `typename` followed by a *qualified-id*
> denotes the type in a non-type
> [Since template *template-parameter*s
> and template *template-argument*s
> are treated as types for descriptive purposes, the terms
> *non-type parameter* and *non-type argument*
> are used to refer to non-type, non-template parameters and arguments.]{.footnote}
> *parameter-declaration*.
> A *template-parameter* of the form
> `class` *identifier* is a *type-parameter*.
> [&mdash; *end note* \]]{.add}
>
> ::: example
>
> ```c++
> class T { /* ... */ };
> int i;
>
> template<class T, T i> void f(T t) {
>   T t1 = i;         // template-parameters @[T]{.tcode}@ and @[i]{.tcode}@
>   ::T t2 = ::i;     // global namespace members @[T]{.tcode}@ and @[i]{.tcode}@
> }
> ```
>
> Here, the template `f` has a *type-parameter*
> called `T`, rather than an unnamed non-type
> *template-parameter* of class `T`.
> :::
>
> A *template-parameter* declaration shall not
> have a *storage-class-specifier*.
> Types shall not be defined in a *template-parameter*
> declaration.
:::
-->

&sect;[temp.param/3]{.sref}

::: std
> [3]{.pnum}
> The *identifier* in a *type-parameter* is not looked up.
> A *type-parameter*
> whose *identifier* does not follow an ellipsis
> defines its *identifier*
> to be a *typedef-name*
> [(if declared without `template`)]{.rm}
> or *template-name*
> [(if declared with `template`)]{.rm}
> in the scope of the template declaration.
> [The *identifier* is a *template-name*
> if it is declared with `template`
> or with a *constraint-specifier* naming a template concept ([temp.concept]{.sref});
> otherwise, it is a *typedef-name*.]{.add}
>
> *[...]*
:::

&sect;[temp.param/4]{.sref}

::: std
> [4]{.pnum}
> A [*type-constraint*]{.rm} [*constraint-specifier*]{.add} `Q`
> that designates a concept `C`
> can be used to constrain a
> contextually-determined type[, template,]{.add} or template [type]{.rm} parameter pack `T`
> with a *constraint-expression* `E` defined as follows.
> If `Q` is of the form `C<A`~1~`,` &ctdot;`, A`~*n*~`>`,
> then let `E`&prime; be `C<T, A`~1~`,` &ctdot;`, A`~*n*~`>`.
> Otherwise, let `E`&prime; be `C<T>`.
> If `T` is not a pack,
> then `E` is `E`&prime;[,]{.rm}[;]{.add}
> otherwise[,]{.add} `E` is `(E`&prime;`@\ @&& ...)`.
> This *constraint-expression* `E` is called the
> *immediately-declared constraint*
> of `Q` for `T`.
> The concept designated by a [*type-constraint*]{.rm} [*constraint-specifier*]{.add}
> shall be a type concept
> [or a template concept]{.add} ([temp.concept]{.sref}).
:::

&sect;[temp.param/5]{.sref}

::: std
> [5]{.pnum}
> A *type-parameter* that starts with a [*type-constraint*]{.rm} [*constraint-specifier*]{.add}
> introduces the immediately-declared constraint
> of the [*type-constraint*]{.rm} [*constraint-specifier*]{.add} for the parameter.
>
> *[...]*
:::

&sect;[temp.param/11]{.sref}

::: std
> [11]{.pnum}
> A non-type template parameter declared with a type that
> contains a placeholder type with a [*type-constraint*]{.rm} [*constraint-specifier*]{.add}
> introduces the immediately-declared constraint
> of the [*type-constraint*]{.rm} [*constraint-specifier*]{.add}
> for the invented type corresponding to the placeholder ([dcl.fct]{.sref}).
:::

&sect;[temp.param/17]{.sref}

::: std
> [17]{.pnum}
> If a *template-parameter* is a
> *type-parameter* with an ellipsis prior to its
> optional *identifier* or is a
> *parameter-declaration* that declares a
> pack ([dcl.fct]{.sref}), then the *template-parameter*
> is a template parameter pack ([temp.variadic]{.sref}).
> A template parameter pack that is a *parameter-declaration* whose type
> contains one or more unexpanded packs is a pack expansion. Similarly,
> a template parameter pack that is a *type-parameter* with a
> *template-parameter-list* containing one or more unexpanded
> packs is a pack expansion.
> A [type]{.rm} [template]{.add} parameter pack [declared]{.add} with a [*type-constraint*]{.rm} [*constraint-specifier*]{.add} that
> contains an unexpanded parameter pack is a pack expansion.
> A template parameter pack that is a pack
> expansion shall not expand a template parameter pack declared in the same
> *template-parameter-list*.
>
> *[...]*
:::

&sect;[temp.names/7]{.sref}

::: std
> [7]{.pnum}
> A *template-id* is *valid* if
> [the named template is a template *template-parameter*
> declared without a *template-head*
> or if]{.add}
>
> - [7.1]{.pnum}
>   there are at most as many arguments as there are parameters
>   or a parameter is a template parameter pack ([temp.variadic]{.sref}),
> - [7.2]{.pnum}
>   there is an argument for each non-deducible non-pack parameter
>   that does not have a default *template-argument*,
> - [7.3]{.pnum}
>   each *template-argument* matches the corresponding
>   *template-parameter* ([temp.arg]{.sref}),
> - [7.4]{.pnum}
>   substitution of each template argument into the following
>   template parameters (if any) succeeds, and
> - [7.5]{.pnum}
>   if the *template-id* is non-dependent,
>   the associated constraints are satisfied as specified in the next paragraph.
>
> A *simple-template-id* shall be valid unless it names a
> function template specialization ([temp.deduct]{.sref}).
>
> *[...]*
:::

&sect;[temp.arg.template/3]{.sref}

::: std
> [3]{.pnum}
> A *template-argument* [`A`]{.add} matches a template
> *template-parameter* `P` when
> [`P` is declared without a *template-head*,
> `A` is the name of a template *template-parameter*
> declared without a *template-head*,
> or when]{.add}
> `P` is at least as specialized as [the *template-argument*]{.rm} `A`[,
> ignoring constraints on `A` if `P` is unconstrained]{.add}.
> [In this comparison, if `P` is unconstrained,
> the constraints on `A` are not considered.]{.rm}
> If `P` [contains]{.rm} [is declared with a *template-head* containing]{.add}
> a template parameter pack [([temp.variadic]{.sref})]{.add}, then `A` also matches `P`
> if each of `A`'s template parameters
> matches the corresponding template parameter in the
> *template-head* of `P` [(this behavior is deprecated; see D.# [\[depr.temp.match\]](#depr.temp.match))]{.add}.
> Two template parameters match if they are of the same kind (type, non-type, template),
> for non-type *template-parameter*s, their types are
> equivalent ([temp.over.link]{.sref}), and for template *template-parameter*s,
> each of their corresponding *template-parameter*s matches, recursively.
> [When `P`'s *template-head* contains a template parameter
> pack ([temp.variadic]{.sref}), the template parameter pack]{.rm}
> [A template parameter pack in the *template-head* of `P`]{.add}
> will match zero or more template
> parameters or template parameter packs in the *template-head* of
> `A` with the same type and form as the template parameter pack in `P`
> (ignoring whether those template parameters are template parameter packs).
>
> ::: example
>
> ```c++
> template<class T> class A { /* ... */ };
> template<class T, class U = T> class B { /* ... */ };
> template<class ... Types> class C { /* ... */ };
> template<auto n> class D { /* ... */ };
> @[`template<template class P> class W { /* ... */ };`]{.add}@
> template<template<class> class @[P]{.rm} [Q]{.add}@> class X { /* ... */ };
> template<template<class ...> class @[Q]{.rm} [R]{.add}@> class Y { /* ... */ };
> template<template<int> class @[R]{.rm} [S]{.add}@> class Z { /* ... */ };
>
> @[`W<A> wa;            // OK`]{.add}@
> @[`W<B> wb;            // OK`]{.add}@
> @[`W<C> wc;            // OK`]{.add}@
> @[`W<D> wd;            // OK`]{.add}@
> X<A> xa;            // OK
> X<B> xb;            // OK
> X<C> xc;            // OK
> Y<A> ya;            // OK
> Y<B> yb;            // OK
> Y<C> yc;            // OK
> Z<D> zd;            // OK
> ```
>
> :::
>
> ::: example
>
> ```c++
> template <class T> struct @[eval]{.rm}@ @[X { }]{.add}@;
>
> template <template <class, class...> class TT, class T1, class... Rest>
> @[`struct eval<TT<T1, Rest...>> { };`]{.rm}@
> @[`using eval1 = TT<T1, Rest...>;`]{.add}@
>
> @[`template <template class TT, class T1, class... Rest>`]{.add}@
> @[`using eval2 = TT<T1, Rest...>;`]{.add}@
>
> template <class T1> struct A;
> template <class T1, class T2> struct B;
> template <int N> struct C;
> template <class T1, int N> struct D;
> template <class T1, class T2, int N = 17> struct E;
>
> @[X<]{.add}@eval@[1]{.add}@<A@[<]{.rm}[,\ ]{.add}@int>> e@[1]{.add}@A;          // OK@[,\ matches\ partial\ specialization\ of\ [eval]{.tcode}]{.rm}@
> @[X<]{.add}@eval@[1]{.add}@<B@[<]{.rm}[,\ ]{.add}@int, float>> e@[1]{.add}@B;   // OK@[,\ matches\ partial\ specialization\ of\ [eval]{.tcode}]{.rm}@
> @[X<]{.add}@eval@[1]{.add}@<C@[<]{.rm}[,\ ]{.add}@17>> e@[1]{.add}@C;           // error: @[C]{.tcode}@ does not match @[TT]{.tcode}@ in @[partial\ specialization]{.rm} [eval1]{.tcode .add}@
> @[X<]{.add}@eval@[1]{.add}@<D@[<]{.rm}[,\ ]{.add}@int, 17>> e@[1]{.add}@D;      // error: @[D]{.tcode}@ does not match @[TT]{.tcode}@ in @[partial\ specialization]{.rm} [eval1]{.tcode .add}@
> @[X<]{.add}@eval@[1]{.add}@<E@[<]{.rm}[,\ ]{.add}@int, float>> e@[1]{.add}@E;   // error: @[E]{.tcode}@ does not match @[TT]{.tcode}@ in @[partial\ specialization]{.rm} [eval1]{.tcode .add}@
>
> @[`X<eval2<A, int>> e2A;           //`]{.add}[\ *OK*]{.add}@
> @[`X<eval2<B, int, float>> e2B;    //`]{.add}[\ *OK*]{.add}@
> @[`X<eval2<C, 17>> e2C;            //`]{.add}[\ *error:\ non-type\ argument\ provided\ for*\ `T1`\ *in*\ `eval2`]{.add}@
> @[`X<eval2<D, int, 17>> e2D;       //`]{.add}[\ *error:\ non-type\ argument\ provided\ for*\ `Rest`\ *in*\ `eval2`]{.add}@
> @[`X<eval2<E, int, float>> e2E;    //`]{.add}[\ *OK*]{.add}@
> ```
>
> :::
>
> ::: example
>
> ```c++
> template<typename T> concept C = requires (T t) { t.f(); };
> template<typename T> concept D = C<T> && requires (T t) { t.g(); };
>
> template<template<C> class P> struct S { };
>
> template<C> struct X { };
> template<D> struct Y { };
> template<typename T> struct Z { };
>
> S<X> s1;            // OK, @[X]{.tcode}@ and @[P]{.tcode}@ have equivalent constraints
> S<Y> s2;            // error: @[P]{.tcode}@ is not at least as specialized as @[Y]{.tcode}@
> S<Z> s3;            // OK, @[P]{.tcode}@ is at least as specialized as @[Z]{.tcode}@
> ```
>
> :::
:::

&sect;[temp.constr.decl/2]{.sref}

::: std
> [2]{.pnum}
> Constraints can also be associated with a declaration through the use of
> [*type-constraint*s]{.rm} [*constraint-specifier*s]{.add}
> in a *template-parameter-list* or *parameter-type-list*.
> Each of these forms introduces additional *constraint-expression*s
> that are used to constrain the declaration.
:::

&sect;[temp.constr.decl/3.3]{.sref}

::: std
>
> - [3.3]{.pnum}
>   Otherwise, the associated constraints are the normal form of a logical
>   [and]{.smallcaps} expression ([expr.log.and]{.sref}) whose operands are in the
>   following order:
>
>   - [3.3.1]{.pnum}
>     the *constraint-expression* introduced by
>     each [*type-constraint*]{.rm} [*constraint-specifier*]{.add} ([temp.param]{.sref}) in
>     the declaration's *template-parameter-list*,
>     in order of appearance, and
>   - [3.3.2]{.pnum}
>     the *constraint-expression* introduced by
>     a *requires-clause* following
>     a *template-parameter-list* ([temp.pre]{.sref}), and
>   - [3.3.3]{.pnum}
>     the *constraint-expression* introduced by
>     each [*type-constraint*]{.rm} [*constraint-specifier*]{.add} in
>     the parameter-type-list of a function declaration, and
>   - [3.3.4]{.pnum}
>     the *constraint-expression* introduced by
>     a trailing *requires-clause* ([dcl.decl]{.sref}) of
>     a function declaration ([dcl.fct]{.sref}).
:::

&sect;[temp.decls.general/3]{.sref}

::: std
> [3]{.pnum}
> For purposes of name lookup and instantiation,
> default arguments,
> [*type-constraint*s]{.rm} [*constraint-specifier*s]{.add},
> *requires-clause*s ([temp.pre]{.sref}),
> and
> *noexcept-specifier*s
> of function templates
> and
> of member functions of class templates
> are considered definitions;
> each
> default argument,
> [*type-constraint*]{.rm} [*constraint-specifier*]{.add},
> *requires-clause*,
> or
> *noexcept-specifier*
> is a separate definition
> which is unrelated
> to the templated function definition or
> to any other
> default arguments,
> [*type-constraint*s]{.rm} [*constraint-specifier*s]{.add},
> *requires-clause*s,
> or
> *noexcept-specifier*s.
> For the purpose of instantiation, the substatements of a constexpr if
> statement ([stmt.if]{.sref}) are considered definitions.
:::

&sect;[temp.over.link/6]{.sref}

::: std
> [6]{.pnum}
> Two *template-head*s are
> *equivalent* if
> their *template-parameter-list*s have the same length,
> corresponding *template-parameter*s are equivalent
> and are both declared with [*type-constraint*s]{.rm} [*constraint-specifier*s]{.add} that are equivalent
> if either *template-parameter*
> is declared with a [*type-constraint*]{.rm} [*constraint-specifier*]{.add},
> and if either *template-head* has a *requires-clause*,
> they both have
> *requires-clause*s and the corresponding
> *constraint-expression*s are equivalent.
> Two *template-parameter*s are
> *equivalent*
> under the following conditions:
>
> - [6.1]{.pnum}
>   they declare template parameters of the same kind,
> - [6.2]{.pnum}
>   if either declares a template parameter pack, they both do,
> - [6.3]{.pnum}
>   if they declare non-type template parameters,
>   they have equivalent types
>   ignoring the use of [*type-constraint*s]{.rm} [*constraint-specifier*s]{.add} for placeholder types, and
> - [6.4]{.pnum}
>   if they declare template template parameters, [either neither has a *template-head* or they both do and]{.add}
>   their [template parameters]{.rm} [*template-head*s]{.add} are equivalent.
>
> When determining whether types or [*type-constraint*s]{.rm} [*constraint-specifier*s]{.add}
> are equivalent, the rules above are used to compare expressions
> involving template parameters.
> Two *template-head*s are
> *functionally equivalent*
> if they accept and are satisfied by ([temp.constr.constr]{.sref})
> the same set of template argument lists.
:::

&sect;[temp.func.order/3]{.sref}

::: std
> [3]{.pnum}
> To produce the transformed template, for each type, non-type, or template
> template parameter (including template parameter packs ([temp.variadic]{.sref})
> thereof) synthesize a unique type, value, or class template
> respectively and substitute it for each occurrence of that parameter
> in the function type of the template[.]{.rm}[:]{.add}
>
> - [[3.1]{.pnum}
>   For a non-type template parameter,
>   the type of the synthesized value
>   is the type of the parameter
>   after substitution
>   of prior template parameters.]{.add}
>
>   ::: note
>   The type replacing the placeholder
>   in the type of the value synthesized for a non-type template parameter
>   is also a unique synthesized type.
>   :::
>
> - [[3.2]{.pnum}
>   For a template template parameter
>   declared with a *template-head*,
>   the *template-head* of the synthesized class template
>   is the result of substitution into the *template-head*
>   of the template template parameter.
>   For a template template parameter
>   declared without a *template-head*,
>   the synthesized class template
>   does not match ([temp.arg.template]{.sref})
>   any template *template-parameter*
>   declared with a *template-head*.
>   ]{.add}
>
> *[...]*
:::

&sect;[temp.concept/2]{.sref}

::: std
> [2]{.pnum}
> A *concept-definition*
> declares a concept.
> Its *identifier* becomes a *concept-name*
> referring to that concept
> within its scope.
> The optional *attribute-specifier-seq* appertains to the concept.
>
> ::: example
>
> ```c++
> template<typename T>
> concept C = requires(T x) {
>   { x == x } -> std::convertible_to<bool>;
> };
>
> template<typename T>
>   requires C<T>     // @[C]{.tcode}@ constrains @[f1(T)]{.tcode}@ in @*constraint-expression*@
> T f1(T x) { return x; }
>
> template<C T>       // @[C]{.tcode}@, as a @[*type-constraint*]{.rm} [*constraint-specifier*]{.add}@, constrains @[f2(T)]{.tcode}@
> T f2(T x) { return x; }
> ```
>
> :::

:::

&sect;[temp.concept/7]{.sref}

::: std
> [7]{.pnum}
> The first declared template parameter of a concept definition is its
> *prototype parameter*.
> A *type concept*
> is a concept whose prototype parameter
> is a type *template-parameter*.
> [A *template concept*
> is a concept whose prototype parameter
> is a template *template-parameter*.]{.add}
:::

&sect;[temp.inst/2]{.sref}

::: std
> [2]{.pnum}
> *[...]*
>
> ::: note
> Within a template declaration,
> a local class ([class.local]{.sref}) or enumeration and the members of
> a local class are never considered to be entities that can be separately
> instantiated (this includes their default arguments,
> *noexcept-specifier*s, and non-static data member
> initializers, if any,
> but not their [*type-constraint*s]{.rm} [*constraint-specifier*s]{.add} or *requires-clause*s).
> As a result, the dependent names are looked up, the
> semantic constraints are checked, and any templates used are instantiated as
> part of the instantiation of the entity within which the local class or
> enumeration is declared.
> :::
:::

&sect;[temp.inst/17]{.sref}

::: std
> [17]{.pnum}
> The [*type-constraint*s]{.rm} [*constraint-specifier*s]{.add} and *requires-clause*
> of a template specialization or member function
> are not instantiated along with the specialization or function itself,
> even for a member function of a local class;
> substitution into the atomic constraints formed from them is instead performed
> as specified in [temp.constr.decl]{.sref} and [temp.constr.atomic]{.sref}
> when determining whether the constraints are satisfied
> or as specified in [temp.constr.decl]{.sref} when comparing declarations.
:::

&sect;[depr]{.sref}

::: {.std .ins}
>
> ### D.# Matching template *template-parameter*s with parameter packs &emsp;&emsp;&emsp;&emsp;&emsp;&emsp;\[depr.temp.match\] {#depr.temp.match}
>
> [1]{.pnum}
> A template parameter pack in the *template-head*
> of a template *template-parameter*
> is permitted to match
> zero or more template parameters
> in the *template-head*
> of a template *template-argument* ([temp.arg.template]{.sref}).
> This behavior is deprecated.
>
> ::: example
>
> ```c++
> template <template <class... Ts> class TT> class X { };
> template <class A, class B> class Y;
>
> X<Y> x;       // deprecated, TT matches A and B
>
> template <template <class A, class B> class TT> class P { };
> template <class... Ts> class Q;
>
> P<Q> p;       // OK
> ```
>
> :::
:::
