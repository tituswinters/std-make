<table border="0" cellpadding="0" cellspacing="0" style="border-collapse: collapse" bordercolor="#111111" width="607">
    <tr>
        <td width="172" align="left" valign="top">Document number:</td>
        <td width="435"><span style="background-color: #FFFF00">DXXXXR0</span></td>
    </tr>
    <tr>
        <td width="172" align="left" valign="top">Date:</td>
        <td width="435">2016-01-15</td>
    </tr>
    <tr>
        <td width="172" align="left" valign="top">Project:</td>
        <td width="435">ISO/IEC JTC1 SC22 WG21 Programming Language C++</td>
    </tr>
    <tr>
        <td width="172" align="left" valign="top">Audience:</td>
        <td width="435">Library Evolution Working Group</td>
    </tr>
    <tr>
        <td width="172" align="left" valign="top">Reply-to:</td>
        <td width="435">Vicente J. Botet Escriba &lt;<a href="mailto:vicente.botet@wanadoo.fr">vicente.botet@wanadoo.fr</a>&gt;</td>
    </tr>
</table>

# A generic `none_t` literal type for *Nullable* types
======================================================

**Abstract**

In the same way we have *NullablePointer* types with `nullptr` to mean a null value, this proposal defines *Nullable* requirements for types for which `none` mean the null value.
This paper propose a generic `none` literal for *Nullable* types, as `optional<T>` and `any` and propose to replace the variant `monostate_t` defined in [P0088R0] by `none_t`. 

Note that for *Nullable* types the null value doesn't mean an error, it is just a value different from all the other values, it is none of the other values.

This takes in account the feedback from Kona meeting [P0032R0]. The direction of the committee was:

* Do we want `none_t` to be a separate paper? 

```
SF F N A SA
11 1 3 0 0```

* Do we want the `operator bool` changes? No, instead a `.something()` member function (e.g. `has_value`) is preferred for the 3 classes. This doesn't mean yet that we replace the existing explicit `operator bool` in `optional`.
 Do we want emptiness checking to be consistent between `any`/`optional`? Unanimous yes

```  Provide operator bool for both  Y:  6 N: 5
  Provide .something()            Y: 17 N: 0
  Provide =={}                    Y:  0 N: 5
  Provide ==std::none             Y:  5 N: 2
  something(any/optional)         Y:  3 N: 8
```

# Table of Contents

1. [Introduction](#introduction)
2. [Motivation and Scope](#motivation-and-scope)
3. [Proposal](#proposal)
4. [Open points](#open-points)
5. [Proposed Wording](#proposed-wording)
6. [Implementability](#implementability)
7. [Acknowledgements](#acknowledgements)
8. [References](#references)

# Introduction

There are currently two adopted no-value specific factories, `nullptr` for pointer-like classes and `nullopt` for `optional<T>`. [P0088R0] proposes `monostate` as a unit type, but without a factory. [P0032R0] proposed a new `none` literal for the class `any`. The feedback from the Kona meeting was that should not keep adding new “unit” types like this and that we need to have a generic `none` literal at least for non pointer-like classes. 

This paper then presents a proposal for a such a generic `none` (no-value) factory, that is able to create the not-a-type class associated to *Nullable* wrapping types.
 
Having a common syntax and semantics for this literal would help to have more readable and teachable code, allows to define generic algorithms that need to create such a no-value instance.
 
Note however that we would not be able to define interesting algorithms without having other functions around the *Nullable* concept as e.g. been able to create a *Nullable* wrapping instance containing the associated value (the make factory [MAKEF]) and observe whether this *Nullable* type contains a value or not (e.g. a visitation type switch as proposed in [P0050], or the getter functions proposed in [P0042], or Functor/Monadic operations).

# Motivation and Scope

## Why do we need a generic `none` literal

There is a proliferation of “unit” types that mean no-value type, 

* `nullptr_t` for pointer-like objects and `std::function`, 
* `std::experimental::nullopt_t` for `optional<T>`,
* `std::experimental::monostate` unit type for `variant<monostate_t, Ts...>` (in ([P0088R0]),
* `none_t` for `any` (in [P0032R0] - rejected as a specific unit type for any) 
 
Having a common and uniform way to name these no-value types associated to *Nullable* types would 
help to make the code more readable and teachable.

Could allows to define generic algorithms that need to create such a no-value instance. 

Generic code working with *Nullable* types, needs a generic way to name the null value. This is the reason d'être of `none_t` and `none`.
 

## Possible ambiguity of a single no-value constant


Before going to far let me show you the current situation with `nullptr` and too my knowledge why `nullptr` was not retained as no-value constant for `optional<T>`. Please correct me if I'm wrong.

### *NullablePointer* types

We have that all the pointer-like types in the standard library are implicitly convertible from and equality comparable to `nullptr_t`. 

```c++
	int* ip = nullptr;
	unique_ptr<int> up= nullptr;
	shared_ptr<int> sp = nullptr;
	ip = nullptr;
	up= nullptr;
	sp = nullptr;
```

Up to now everything is ok. We have the needed context to avoid ambiguities. 

However, if we have an overloaded function as e.g. print

```c++
	template <class T>
	void print(unique_ptr<T> ptr);
	template <class T>
	void print(shared_ptr<T> ptr);
```

The following call would be ambiguous

```c++
	print(nullptr);
```

Wait, who wants to print `nullptr`? Surely nobody. Anyway we could add an overload for `nullptr_t`

```c++
	void print(nullptr_t ptr);
```

and now the last overload will be preferred as there is no need to conversion.

If we want however to call to a specific overload we need to build the specific pointer-like type, e.g if wanted the `shared_ptr<T>` overload, we will write

```c++
	print(shared_ptr<int>{});
```

Note that the last call contains more information than should be desired. The `int` type is in some way redundant. It would be great if we could give as less information as possible as in


```c++
	print(nullptr<shared_ptr>));
```

Clearly the type for `nullptr<shared_ptr>` couldn't be `nullptr_t`, nor a specific `shared_ptr<T>`. So the type of `nullptr<shared_ptr>` should be something differen, let me call it e.g. `nullptr_c_t<shared_ptr>`

You can read `nullptr<shared_ptr>` as the nul pointer value associated to `shared_ptr`.

Note that even if template parameter deduction for constructors [P0091R0] is adopted we are not able to write as the deduced type will not be the expected one.

```c++
	print(shared_ptr(nullptr));
```
 
We are not proposing these for `nullptr` in this paper, it is just to present the context. To the authors knowledge it has been accepted that  the user need to be as explicit as needed.

```c++
	print(shared_ptr<int>{});
```

## WHy `nullopt` was introduced?

Lets continue with `optional<T>`. Why the committee didn't wanted to reuse `nullptr` as no-value for `optional<T>`?

	optional<int> oi = nullptr;
	oi = nullptr;

I believe that the two main concerns were that `optional<T>` is not a pointer-like type even it it defines all the associated operations and that having an `optional<int*>` the following would be ambiguous, 

	optional<int*> sp = nullptr;

	
We need a different type that can be used either for all the *Nullable* types or for those that are wrapping an instance of a type, not pointing to that instance. At the time, as the problem at hand was to have an `optional<T>`, it was considered that a specific solution will be satisfactory. So now we have

```c++	
	template <class T>
	void print(optional<T> o);

	optional<int> o = nullopt;
	o = nullopt;
	print(nullopt);
```

## Moving to *Nullable* types

Some could think that it is better to be specific. But what would be wrong having a single way to name this no-value for a specific class using `none`?

```c++
	optional<int> o = none;
	any a = none;
	o = none;
	a = none;
```

As far as the context is clear there is no ambiguity.

We could add as well the overload to `print` the no-value none

```c++
	void print(none_t);
```

and 

```c++
	print(none);
	print(optional<int>{});
```


So now we can see `any` as a *Nullable* if we provide the conversions from `none_t`

```c++
	any a = none;
	a = none;
	print(any{});
```

## Nesting *Nullable* types

We don't provide a solution to the following use case. How to initialize an `optional<any>` with an `any`  none

```c++
optional<any> = none;
```

```c++
optional<any> = any{};
```

Note that `any` is already `Nullable`, so how will this case be different from 

```c++
optional<optional<int>> = optional<int>{};
```

Not proposed by this paper, would be the possibility to lift `none`. This lifted value would express explicitly that the wrapped value would be used to emplace the optional wrapped valued. 

```c++
optional<any> o = lift(none);
```

The result of lift would be a type that will wrap `none_t`. `optional<T>` will need to accept a conversion from this `lifted<U>` by emplacing the type `T` from the type `U`.


## Other operations with no-value type

There are other operations between the wrapping type and the not a value type, as the mixed equality comparison.

```c++
	o == nullopt;
	a == any{};
```

if we define these operations we will be able to have that the following expressions will be well formed.

```c++
	o == none
	a == none
	none != o
	none != a
```

Type erased classes as `std::experimental::any` don't provide order comparison.


However *Nullable* types wrapping a type as `optional<T>` provide mixed comparison if the type `T` is ordered. 

```c++
	o > none
	o >= none
	! (o < none)
	! (o <= none)
```

A possible `nullable_variant<Ts..>` could provide mixed comparison between the `none_t` type and the *Nullable* `nullable_variant<Ts..>` type itself. 

So the question is if we can define these mixed comparison once for all for a generic `none_t` type and a model of *Nullable*.

```c++
	template < Nullable C >
	bool operator==(none_t, C const& x) { return ! x.has_value(); }
	template < Nullable C >
	bool operator==(C const& x, none_t { return ! x.has_value(); }
	template < Nullable C >
	bool operator!=(none_t, C const& x) { return x.has_value(); }
	template < Nullable C >
	bool operator!=(C const& x, none_t) { return x.has_value(); }

```

The ordered comparison operations could defined only if the *Nullable* class is Ordered.

# Proposal

This paper propose to 

* add `none_t`/`none`,
* add requirements for *Nullable* and *StrictWeaklyOrderedNullable* types, and derive the mixed comparison operations,
* some minor changes to `optional`, `any` and `variant` to take `none_t` as its no-value type.

## But how `none` is defined?

This proposal doesn't impose a specific implementation. The `std::experimental::none` is a user defined literal that is the single value of the type, `std::experimental::none_t`. We say that `none_t` is a unit type.

A possible implementation could be based on the current definition of `std::experimental::nulopt_t` and `std::experimental::nulopt`.  

 
# Proposed WordingThe proposed changes are expressed as edits to [N4564] the Working Draft - C++ Extensions for Library Fundamentals V2. 

## Nullable Objects

### No-value state indicator

The `std::experimental::none` is a user defined literal that is the single value of the type, `std::experimental::none_t`. We say that `none_t` is a unit type.

`std::experimental::none_t`  shall be a literal type. Constant `none` shall be initialized with an argument of literal type.

[Note: `std::experimental::none_t` is a distinct unit type to indicate the state of not containing a value for *Nullable* objects. The single value of this type `none` is a constant that can be converted to any *Nullable* type and that must equally compare to a default constructed *Nullable*. —- endnote]

### *Nullable* requirements

A *Nullable* type is a type that supports a distictive null value. A type `N` meets the requirements of *Nullable* if:
* `N` satisfies the requirements of *DefaultConstructible*, and *Destructible*,* the expressions shown in the table below are valid and have the indicated semantics, and* N satisfies all the other requirements of this subclause.A value-initialized object of type `N` produces the null value of the type. The null value shall be equivalent only to itself. A default-initialized object of type `N` may have an indeterminate value. [ Note: Operations involving indeterminate values may cause undefined behavior. — end note ]

No operation which is part of the *Nullable* requirements shall exit via an exception.In Table below, `u` denotes an identifier, `t` denotes a non-const lvalue of type `N`, `x` denotes a (possibly const) expression of type `N`, 
and `n` denotes a value of type (possibly const) `std::experimental::none_t`.

<table border="0" cellpadding="0" cellspacing="0" style="border-collapse: collapse" bordercolor="#111111" width="607">
    <tr>
        <td align="left" valign="top"> Expression </td>
        <td align="left" valign="top"> Return Type </td>
        <td align="left" valign="top"> Operational Semantics </td>
    </tr>
    <tr>
        <td align="left" valign="top"> N u(n) </td>
        <td align="left" valign="top"> </td>
        <td align="left" valign="top"> post: u == N{} </td>
    </tr>
    <tr>
        <td align="left" valign="top"> N u = n </td>
        <td align="left" valign="top"> </td>
        <td align="left" valign="top"> post: u == N{} </td>
    </tr>
    <tr>
        <td align="left" valign="top"> t = n </td>
        <td align="left" valign="top"> N&</td>
        <td align="left" valign="top"> post: t == N{} </td>
    </tr>
    <tr>
        <td align="left" valign="top"> x.has_value() </td>
        <td align="left" valign="top"> contextualy convertible to bool</td>
        <td align="left" valign="top"> x != N{} </td>
    </tr>

</table>

Mixed equality comparaison between a *Nullable* and a `none_t` are defined as

```c++
	template < Nullable C >
	bool operator==(none_t, C const& x) { return ! x.has_value(); }
	template < Nullable C >
	bool operator==(C const& x, none_t { return ! x.has_value(); }
	template < Nullable C >
	bool operator!=(none_t, C const& x) { return x.has_value(); }
	template < Nullable C >
	bool operator!=(C const& x, none_t) { return x.has_value(); }

```


### *StrictWeaklyOrderedNullable* requirements

A type `N` meets the requirements of *StrictWeaklyOrderedNullable* if:
* `N` satisfies the requirements of *StrictWeaklyOrdered* and *Nullable*.


Mixed ordered comparaison between a *StrictWeaklyOrderedNullable* and a `none_t` are defined as

```c++
	template < StrictWeaklyOrderedNullable C >
	bool operator<(none_t, C const& x) { return x.has_value(); }
	template < StrictWeaklyOrderedNullable C >
	bool operator<(C const& x, none_t { return false; }
	
	template < StrictWeaklyOrderedNullable C >
	bool operator<=(none_t, C const& x) { return true; }
	template < StrictWeaklyOrderedNullable C >
	bool operator<=(C const& x, none_t { return ! x.has_value(); }

	template < StrictWeaklyOrderedNullable C >
	bool operator>(none_t, C const& x) { return false; }
	template < StrictWeaklyOrderedNullable C >
	bool operator>(C const& x, none_t { return x.has_value(); }
	
	template < StrictWeaklyOrderedNullable C >
	bool operator>=(none_t, C const& x) { return ! x.has_value(); }
	template < StrictWeaklyOrderedNullable C >
	bool operator>=(C const& x, none_t { return true; }
```



### Header <experimental/nullable> synopsis [nullable.synop]

```c++
namespace std {
  namespace experimental {
  inline namespace fundamentals_v2 {
  
    none // unspecified constant;
    using none_t = decltype(none);

    // Comparison with none_t
    template < Nullable C >
      bool operator==(none_t, C const& x) noexcept { return ! x.has_value(); }
    template < Nullable C >
      bool operator==(C const& x, none_t) noexcept { return ! x.has_value(); }
    template < Nullable C >
      bool operator!=(none_t, C const& x) noexcept { return x.has_value(); }
    template < Nullable C >
	   bool operator!=(C const& x, none_t) noexcept { return x.has_value(); }

    template < StrictWeaklyOrderedNullable C >
	   bool operator<(none_t, C const& x) { return x.has_value(); }
    template < StrictWeaklyOrderedNullable C >
	   bool operator<(C const& x, none_t { return false; }
	template < StrictWeaklyOrderedNullable C >
	   bool operator<=(none_t, C const& x) { return true; }
    template < StrictWeaklyOrderedNullable C >
	   bool operator<=(C const& x, none_t { return ! x.has_value(); }
    template < StrictWeaklyOrderedNullable C >
	   bool operator>(none_t, C const& x) { return false; }
    template < StrictWeaklyOrderedNullable C >
	   bool operator>(C const& x, none_t { return x.has_value(); }
    template < StrictWeaklyOrderedNullable C >
	   bool operator>=(none_t, C const& x) { return ! x.has_value(); }
    template < StrictWeaklyOrderedNullable C >
	   bool operator>=(C const& x, none_t { return true; }
  }
  }
}
```


## Optional Objects

**Add** `optional<T>` is a model of *NullableValue*.

**Add** `optional<T>` is a model of *StrictWeaklyOrderedNullable* if `T` is a model of *StrictWeaklyOrdered*.

**Remove** the definiton of `nullopt_t`/`nullopt`.
 
**Replace** any use of `nullopt_t`/`nullopt` by `none_t`/`none`.

**Remove** the mixed operations as redondant [optional.nullops].

## Class Any

**Add** `any` is a model of *NullableValue*.

**Add** a constructor from `none_t` equivalent to the default constructor.

**Add** a assignemnt from `none_t` equivalent assigning a default constructed object.

## Variant Object

Waiting for a specific wording in the TS.

**Remove** the definiton of `monostate_t`.

**Replace** any additional use of `monostate_t` by `none_t`.

 
# Implementability

This proposal can be implemented as pure library extension, without any compiler magic support, in C++14.

# Open points
 
The authors would like to have an answer to the following points if there is at all an interest in this proposal:

* Should we include `none` in `<experimental/functional>` or in a specific file? 
	* We believe that a specific file is a better choice as this is needed in `<experimental/optional>`, `<experimental/any>` and `<experimental/variant>`. I propose `<experimental/none>`.

* Should the mixed comparison with `none_t` be defined implicitly? 

    * An alternative is to don't define them. In this case it could be better to remove the *Nullable* and *StrictWeaklyOrderedNullable* requirements as the reason d'être of those requirements is to define these operations.


# Acknowledgements

Thanks to Tony Van Eerd for championing this proposal during the C++ standard committee meetings and helping me to improve globally the paper. Thanks to Agustín Bergé K-ballo for his useful comments.

# References

[N4564]: http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/n4564.pdf "N4564 - Working Draft, C++ Extensions for Library Fundamentals, Version 2 PDTS"[P0032R0]: http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/p0032r0.pdf "Homogeneous interface for variant, any and optional"[P0088R0]: http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/p0088r0.pdf "Variant: a type-safe union that is rarely invalid (v5)"

    [P0091R0]: http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/p0091r0.html "Template parameter deduction for constructors (Rev. 3)"

* [N4564] N4564 - Working Draft, C++ Extensions for Library Fundamentals, Version 2 PDTS

    http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/n4564.pdf* [P0032R0] Homogeneous interface for variant, any and optional

    http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/p0032r0.pdf* [P0091R0] Template parameter deduction for constructors (Rev. 3)

    http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/p0091r0.html

* [P0088R0] Variant: a type-safe union that is rarely invalid (v5)

    http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/p0088r0.pdf