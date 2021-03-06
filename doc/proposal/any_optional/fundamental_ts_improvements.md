<table border="0" cellpadding="0" cellspacing="0" style="border-collapse: collapse" bordercolor="#111111" width="607">
    <tr>
        <td width="172" align="left" valign="top">Document number:</td>
        <td width="435"><span style="background-color: #FFFF00">DXXXX</span>=yy-nnnn</td>
    </tr>
    <tr>
        <td width="172" align="left" valign="top">Date:</td>
        <td width="435">2015-09-07</td>
    </tr>
    <tr>
        <td width="172" align="left" valign="top">Project:</td>
        <td width="435">Programming Language C++, Library Evolution Working Group</td>
    </tr>
    <tr>
        <td width="172" align="left" valign="top">Reply-to:</td>
        <td width="435">Vicente J. Botet Escriba &lt;<a href="mailto:vicente.botet@wanadoo.fr">vicente.botet@wanadoo.fr</a>&gt;</td>
    </tr>
</table>

Adding Coherency Between `variant<Ts...>`,  `any` and `optional<T>`
===============================================

# Introduction

This paper identifies some minor inconveniences in the design of `variant<Ts...>`,  `any` and `optional`, 
diagnoses them as owing to unnecessary asymmetry between those classes, 
and proposes wording to eliminate the asymmetry (and thus the inconveniences). 

The identified issues are related to the last Fundamental TS proposal [n4335] (https://github.com/viboes/std-make/blob/master/doc/proposal/any_optional/fundamental_ts_improvements.md#n4335) and the variant proposal [n4542] (https://github.com/viboes/std-make/blob/master/doc/proposal/any_optional/fundamental_ts_improvements.md#n4542) and  concerns mainly:
* coherency of functions that behave the same but that are named differently,
* addition of emplace functions for `any` class,
* addition of emplace factories for `any` and `optional` classes.
* replacement of `emplace_type<T>`/`emplace_index<I>` by `in_place<T>`/`in_place<I>`
* replacement of the proposed variant `get/get_if` interbace by the something like the `any_cast`, `variant_cast`
* replacement of `bad_optional_access` by `bad_optional_cast`
* replacement of `bad_variant_access` by `bad_optional_cast`
* make `bad_optional_cast` and `bad_variant_cast` inherit from `bad_cast`

# Motivation and Scope

Both `optional` and `any` are classes that can store possibly some underlying type. In the case of `optional` the underlying type is know at compile time, for `any` the underlying type is any and know at run-time.
If `variant<Ts...>`proposal ends been possibly empty, the stored type is any of the `Ts` and know at run-time.

The following incoherencies have been identified:
* `variant<Ts...>` and `optional` provides in place construction while `any` requires a specific instance.

* `variant<Ts...>` and `optional` provides emplace assignment while `any` requires a specific instance to be assigned. 
 
* The in place tags for `variant<Ts...>` and `optional` are different, and we don't know how they could be the same. However the name should be as close as possible.

* `any` provides a `any::clear()` to unset the value while `optional` uses assignment from a `nullopt_t`. If `variant<Ts...>`proposal ends been possibly empty, we expect that it will have a `reset()` member function.

* `optional` provides a `explicit bool` conversion while `any` provides an `any::empty` member function. If `variant<Ts...>` proposal ends been possibly empty, we expect that it will have a  `explicit bool` conversion.

* `optional<T>`, `variant<Ts...>` and `any` provides different interface to get the stored value. `optional` uses a `value` member function, `variant` uses a tuple like interface, while `any` uses a cast like interface. As all these classes are in someway sum types, the first two limited and know at compile time, the last unlimited, it seems natural that both provide the same kind of interface. In addition it seems natural that the exception thrown when the access/cast fails inherits from a common exception.

The C++ standard should be coherent for features that behave the same way on different types. 
Instead of creating specific issues, we have preferred to write a specific paper so that 
we can discuss of the whole view.

# Proposal

We propose to: 

* Add `in_place` a template function (see [eggs::variant](https://github.com/eggs-cpp/variant)).

* In class `optional<T>`
  * Add a `reset` member function.

* Add an `optional_cast` factory.

* Replace `bad_optional_access` by `bad_optional_cast` and make it inherit from `bad_cast`.
 
* Add an `emplace_optional` factory.

* In class `any`
  * make the default constructor constexpr,
  * add in place forward constructors,
  * add emplace forward member functions,
  * rename the `empty` function with an `explicit bool` conversion,
  * rename the `clear` member function to `reset`,

* Add a `none` constexpr variable of type `any` (or type `none_t`).

* Add an `emplace_any` factory.
  
* In class `variant<T>`
  * Replace the uses of `emplace_type_t<T>`/`emplace_index_t<I>` by `in_place_t (&)(unspecified<T>)`/`in_place_t (&)(unspecified<I>)` 
  * Replace the uses of `emplace_type<T>`/`emplace_index<I>` by `in_place<T>`/`in_place<I>` 
  * If `variant<Ts...>` proposal ends been possibly empty, 
    * Add a `reset` member function.
    * Add an `explicit bool` conversion
  * Replace the `get<T>(variant<Ts...>)` by `variant_cast<T>(variant<Ts...>)`.
  * Replace the `get<I>(variant<Ts...>)` by `variant_cast<I>(variant<Ts...>)`
  * Replace the `get_if<T>(variant<Ts...>)` by `variant_cast<T>(variant<Ts...>*)`.
  * Replace the `get_if<I>(variant<Ts...>*)` by `variant_cast<I>(variant<Ts...>*)`

* Replace `bad_variant_access` by `bad_variant_cast` and make it inherit from `bad_cast`.
 
# Design rationale

## `any` in_place constructor

`optonal<T>` in place constructor constructs implicitly a `T`. 

```c++
template <class... Args> 
constexpr explicit optional<T>::optional(in_place_t, Args&&... args);
```

In place construct for `any` can not have an implicit type `T`. We need a way to state explicitly which `T` must be constructed in place.
The function `in_place_t(&)(unspecified<T>)` is used to convey the type `T` participating in overload resolution.

```c++
template <class T, class ...Args>
any(in_place_t(&)(unspecified<T>), , Args&& ...);
```

This can be used as 

```c++
any(in_place<X>, v1, ..., vn);
```

Adopting this template class to optional would needs to change the definition of `in_place` to 

```c++
constexpr in_place_t in_place(unspecified) { return {} };
```

and 

```c++
template <class... Args> 
constexpr explicit optional<T>::optional(in_place_t (&)(unspecified), Args&&... args);
```

Fortunatelly using function references would work for any unary function taken the unspecified type and returning `in_place_t` in addition to `in_place`. Of course defining such a function wouldimply to hack thi unspecified type. This can be seen as a hole on this proposal, but the author think that it is better to have a uniform interface than protecting from malicious attacks from a haler.


## `any` emplace forward member function

`optional<T>` emplace member function emplaces implicitly a `T`. 

```c++
template <class ...Args>
optional<T>::emplace(Args&& ...);
```

`emplace` for `any` can not have an implicit type `T`. 
We need a way to state explicitly which `T` must be emplaced. 

```c++
template <class T, class ...Args>
any::emplace(Args&& ...);
```

and used as follows

```c++
any a;
a.emplace<T>(v1, ..., vn);
```

## About `empty()`/`explicit operator bool()` member functions

`empty` is more associated to containers. We don't see neither `any` nor `optional` as container classes. 
For probably valued types (as are the smart pointers and optional) the standard uses explicit bool conversion instead.  
We consider `any` as a probably valued type.

## About `clear()/reset()` member functions

`clear` is more associated to containers. We don't see neither `any` nor `optional` as container classes. 
For probably valued types (as are the smart pointers) the standard uses `reset` instead.  

## About the constant `none`

Instead of an additional `none`, `any{}` plays well the role. 
However, the authors think that using `none` is much more explicit.

```c++
any a = 1;
a = none;
```

## Do we need a specific type for `none` 

Instead of an additional type `none_t` to declare the constant `none` as `constexpr none_t none{}`,
we can just declare it as `constexpr any none{}` which plays well its role. 

The advantages of the `any` constant is that we don't need conversions.
However, assignment from `none` could be less efficient. 
If performance is required the user should use the `reset` function.

Alternatively we can add `none_t` and define the corresponding conversions.

## About a `none_t` type implicitly convertible to `any` and `optional` 

An alternative to the reset member function would be to be able to assign a `none_t` to an `optional` and to an `any`. 
We could consider that a default instance of `any` contains an instance of a `none_t` type, as `optional` contains a `nullopt`.

The problem is that then `none` can be seen as an `optional` or an `any` and could result in posible ambiguity.

We think that this implicit conversions go against the raison d'être of `nullptr` and that we need explicit factories/constants.

## Do we need an explicit `make_any` factory? 

`any` is not a generic type but a type erased type. `any` play the same role than a posible `make_any`.

This is way this paper doesn't propose a `make_any` factory. 

## About an emplace factories

However, we could consider an emplace_xxx factory that in place constructs a `T`.

`optional<T>` and `any` could be in place constructed as follows:

```c++
optional<T> opt(in_place_t(&)(unspecified), v1, vn);
f(optional<T>(in_place, v1, vn));

any a(in_place_t(&)(unspecified<T>), v1, vn);
f(any(in_place<T>, v1, vn));
```

When we use auto things change a little bit

```c++
auto opt=optional<T>(in_place, v1, vn);
auto a=any(in_place<T>, v1, vn);
```

This is not uniform. Having an `emplace_xxx` factory function would make the code more uniform

```c++
auto opt=emplace_optional<T>(v1, vn);
f(emplace_optional<T>(v1, vn));

auto a=emplace_any<T>(v1, vn);
f(emplace_any<T>(v1, vn));
```

The implementation of `emplace_any` could be:

```c++
template <class T, class ...Args>
emplace_any(Args&& ...args) {
    return any(in_place<T>, std::forward<Args>(args)...);
}
```

It is possible to replace `emplace_optional` by an overloaded extension of `make_optional`. 
We have an implementation of this kind of overload in [GF](https://github.com/viboes/std-make/blob/master/doc/proposal/any_optional/fundamental_ts_improvements.md#c-generic-factory---implementation) as part of the  [DXXXX](https://github.com/viboes/std-make/blob/master/doc/proposal/any_optional/fundamental_ts_improvements.md#dxxxx) proposal. 

The authors prefer the overloaded version, however this is not part of this proposal.

```c++
auto opt=emplace_optional<T>(v1, vn);
f(make_optional<T>(v1, vn));

auto a=emplace_any<T>(v1, vn);
f(make_any<T>(v1, vn));
```

## Which file for `in_place_t` and `in_place`?

As `in_place_t` and `in_place` are used by `optional` and `any` we need to move its definition to another file.
The preference of the authors will be to place them in `<experimental/utility>`.

Note that `in_place`can also be used by `experimental::variant` and that in this case it could also take an index as template parameter.

## Getters versus cast

The generic `get<T>(t)` and `get<I>(t)` is convenient for prodcut types as we know that the product type will contain an instance of any one of its parts. However, both `any` and `variant` are sum types, andso we are not sure the sumtype stores the request type. This is why `any` propose the useof `any-cast`. We suggest that variant sould usesome kind of cast, e.g. `variant_cast`, and in the same way we have `get` for product types, why not have the same generic name for sum types `sum_cast`.      

Moving to a cast like interface goes together with changing of`bad_xxx_access` to `bad_xxx_cast` both for `optional` and `variant`.


# Open points

* Do we want in place constructor for `any`?

* Do we want to adopt the new `in_place` definition ? 

* Do we want the `clear` and `reset` changes?

* Do we want the `operator bool` changes?

* Do we want the `emplace_xxx` factories?

* Do we want to move `variant` access interface from `get`/`get_if` to a cast like `variant_cast` interface?

* If yes, do we want a generic `sum_cast`? 

* If yes, do we want a generic `sum_cast` overload for `optional`?  

# Technical Specification

The wording is relative to [n4480] (https://github.com/viboes/std-make/blob/master/doc/proposal/any_optional/fundamental_ts_improvements.md#n4480).

The present wording doesn't contain any modification to the variant proposal, as it is not yet on the TS. 

Move `in_place_t` from [optional/synop] and [optional/inplace] to the <experimental/utility> synopsis, replace  `in_place` by`

```c++
struct in_place_t {};
constexpr in_place_t in_place(unspecified);
template <class ...T>;
constexpr in_place_t in_place(unspecified<T...>);
template <size N>;
constexpr in_place_t in_place(unspecified<N>);
```


Update [optional.synopsis] adding after `make_optional`

```c++
  template <class T, class ...Args>
    optional<T> emplace_optional(Args&& ...args);
```

Update  [optional.object] updating `in_place_t` by  `in_place_t (&)(unspecified)` and add

```c++
    void reset() noexcept;
```

Add in [optional.specalg]

```c++
  template <class T, class ...Args>
    optional<T> emplace_optional(Args&& ...args);
```

  *Returns*:
    optional<T>(in_place, std::forward<Args>(args)...). 



Update [any.synopsis] adding

```c++
  template <class T, class ...Args>
    any emplace_any(Args&& ...args);
```

Add inside class `any`

```c++
    template <class T, class ...Args>
      any(in_place_t (&)(unspecified<T>), Args&& ...);
    template <class T, class U, class... Args>
      explicit any(in_place_t (&)(unspecified<T>), initializer_list<U>, Args&&...);
    
    template <class T, class ...Args>
      void emplace(Args&& ...);
    template <class T, class U, class... Args>
      void emplace(initializer_list<U>, Args&&...);
```

Replace inside class `any`

```c++
    void clear() noexcept;
    bool empty() const noexcept;
```

by

```c++
    void reset() noexcept;
    explicit operator bool() const noexcept;
```

Add after class `any`

  constexpr any none{};

Add in [any/cons]  `any` construct/destruc after p14

```c++
    template <class T, class ...Args>
    any(in_place_t(&)(unspecified<T>), Args&& ...);   
  };
```

  *Requires*:
    is_constructible_v<T, Args&&...> is true.
  *Effects*:
    Initializes the contained value as if direct-non-list-initializing an object of type 
    `T` with the arguments `std::forward<Args>(args)...`.
  *Postconditions*:
    *this contains a value of type `T`.
  *Throws*:
    Any exception thrown by the selected constructor of `T`.


```c++
    template <class T, class U, class ...Args>
    any(in_place_t (&)(unspecified<T>), initializer_list<U> il, Args&& ...args);   
  };
```

  *Requires*:
    `is_constructible_v<T, initializer_list<U>&, Args&&...>` is true.
  *Effects*:
    Initializes the contained value as if direct-non-list-initializing an object of type `T` with the arguments `il, std::forward<Args>(args)...`.
  *Postconditions*:
    `*this` contains a value.
  *Throws*:
    Any exception thrown by the selected constructor of `T`.
  *Remarks*:
    The function shall not participate in overload resolution unless `is_constructible_v<T, initializer_list<U>&, Args&&...>` is true. 
    

Add in [any/modifiers] `any` modifiers

```c++
    template <class T, class ...Args>
    void emplace(Args&& ...);
  };
```

  *Requires*:
    is_constructible_v<T, Args&&...> is true.
  *Effects*:
    Calls `this.reset()`. Then initializes the contained value as if direct-non-list-initializing 
    an object of type `T` with the arguments `std::forward<Args>(args)...`.
  *Postconditions*:
    *this contains a value.
  *Throws*:
    Any exception thrown by the selected constructor of `T`.
  *Remarks*:
    If an exception is thrown during the call to `T`'s constructor, 
    `*this` does not contain a value, and the previous (if any) has been destroyed. 

```c++
    template <class T, class U, class ...Args>
    void emplace(initializer_list<U> il, Args&& ...);
  };
```

  *Effects*:
    Calls this->reset()`. 
    Then initializes the contained value as if direct-non-list-initializing 
    an object of type `T` with the arguments `il, std::forward<Args>(args)...`.
  *Postconditions*:
    `*this` contains a value.
  *Throws*:
    Any exception thrown by the selected constructor of `T`.
  *Remarks*:
    If an exception is thrown during the call to `T`'s constructor, 
    `*this` does not contain a value, and the previous (if any) has been destroyed.

    The function shall not participate in overload resolution unless `is_constructible_v<T, initializer_list<U>&, Args&&...>` is true.

Replace in [any/modifier], `clear` by `reset`.

Replace in [any/observers], `empty` by `explicit operator bool.

Add in [any/nonmember]

```c++
  template <class T, class ...Args>
    any emplace_any(Args&& ...args);
```

  *Returns*:
    any(in_place<T>, std::forward<Args>(args)...). 
   
# Acknowledgements 

Thanks to Jeffrey Yasskin to encourage me to report these as possible issues of the TS, 
Agustin Bergé for his suggestions about `none` and `in_place`.  
Giovanni Pietro Dereta for its comments concerning `in_place`

# References

## n4480
Working Draft, C++ Extensions for Library Fundamentals 
http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/n4480.html

## n4542
Variant: a type-safe union (v4) 
http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/n4542.pdf

## DXXXX - C++ generic factory 
https://github.com/viboes/std-make/blob/master/doc/proposal/factories/DXXXX_factories.md

## C++ generic factory - Implementation 
https://github.com/viboes/std-make/blob/master/include/experimental/std_make_v1/make.hpp

## eggs::variant
https://github.com/eggs-cpp/variant



