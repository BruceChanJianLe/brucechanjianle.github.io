---
title: C++ Structured Bindings (choosing the right qualifier)
author: Chan Jian Le
date: 2026-08-24 21:18:42 +0800
categories: [Programming]
tags: [programming, cpp]
---

> Notes on structured bindings, specifically on the part that trips most of us
> up - deciding between `auto`, `auto&`, `const auto&` and `auto&&`. If you have
> ever written `auto&&` simply because it felt safest, this article is for you.

Since C++17, most of us have happily replaced `std::tie` and `.first`/`.second`
with structured bindings:

```cpp
for (const auto& [key, value] : my_map)
{
  std::cout << key << " -> " << value << "\n";
}
```

It reads well and nobody complains during code review. But I always wonder
internally, when and why should I use two `&&` v.s. `&` or even none at all.
I have seen `auto&&` sprinkled everywhere as a "just in case" hedge, and I have
also seen `auto [a, b]` quietly copying a 200 byte struct inside a hot loop.

So let's dive deeper!

## The Mental Model

Here is the single most useful thing to internalise. A structured binding does
**not** declare `a` and `b`. It declares a hidden variable (let's call it `e`),
and `a` and `b` become *aliases* into that hidden variable.

```cpp
auto&& [a, b] = f();

// Roughly equivalent to:
//   auto&& e = f();     // the auto / & / && / const applies HERE
//   a, b are just names for e's members - no copies, no extra indirection
```

The qualifier we write belongs to `e`, not to `a` and `b`.

Which means the question we should be asking is not some new structured-binding
specific question. It is the exact same question we have been answering for
years with plain `auto`:

**Do I want a copy, a read-only view, a mutable view, or "whatever it is"?**

## The Defaults

| Intent | Use |
| --- | --- |
| I want my own copy, or the source is a temporary I own | `auto [a, b]` |
| Read-only view of something that already exists | `const auto& [a, b]` |
| I intend to modify the underlying object | `auto& [a, b]` |
| Generic code, or source may be a prvalue *or* an lvalue | `auto&& [a, b]` |

In practice:

```cpp
// Cheap prvalue that we own - just take it
auto [iter, inserted] = map.try_emplace(key, value);

// Reading
for (const auto& [key, val] : m)
{
  use(key, val);
}

// Writing
for (auto& [key, val] : m)
{
  val *= 2;
}
```

ATTENTION: `auto [a, b] = big_struct_lvalue;` copies the **entire** object,
even when we only ever touch one member. This is a surprisingly common
accidental cost, because the copy is invisible at the call site. There is no
`=` on the right hand side to make us suspicious, and the brackets make it look
like we are only pulling out the two members we named.

## When auto&& Genuinely Earns Its Place

So if `const auto&` and `auto&` cover the everyday cases, when is `auto&&`
actually the right answer rather than a stylistic hedge?

The honest answer is: when the range hands us a **prvalue proxy** instead of a
real reference. This shows up immediately once we start using the ranges
library.

```cpp
for (auto&& [i, x] : v | std::views::enumerate)
{
  // ...
}

for (auto&& [a, b] : std::views::zip(xs, ys))
{
  a += b;
}
```

Let's walk through why the alternatives fall short here:

- `auto&` does not compile. `enumerate` and `zip` yield a *tuple by value*, a
  prvalue, and we cannot bind a non-const lvalue reference to a prvalue.
- `const auto&` compiles, but it makes the **proxy** const, which is quite
  misleading. The tuple is const, however the references living inside it are
  not, so we can often still mutate the underlying elements through it. Const
  that does not actually mean const is worse than no const at all.
- `auto&&` handles the lvalue-reference case and the prvalue case uniformly,
  and lifetime extension applies to the hidden variable `e`, so the temporary
  tuple stays alive for the whole loop body.

That second point is worth seeing rather than being told, because it looks so
wrong:

```cpp
std::vector<int> xs{1, 2, 3}, ys{10, 20, 30};

for (const auto& [a, b] : std::views::zip(xs, ys))
{
  a += b;      // compiles happily, and really does modify xs
}

for (auto v : xs) std::cout << v << " ";   // 11 22 33
```

The `const` landed on the `std::pair<int&, int&>` proxy, and a const pair of
non-const references still lets us write through those references. Meanwhile
swapping that same loop to `auto&` fails outright with *cannot bind non-const
lvalue reference of type `std::pair<int&, int&>&` to an rvalue*. Neither
qualifier is telling us the truth about what the loop does.

The same reasoning applies to any proxy-like type. `std::vector<bool>` and its
famous `reference` type, custom reference wrappers, and templates where we
simply do not know what the range will yield.

That last one is the key point. `auto&&` is the correct choice in **generic**
code precisely because we cannot answer the copy/read/write question at the
point of writing. Somebody else answers it later when they instantiate our
template. Outside of that situation, we do know the answer, so we should say it.

## The Trap - auto&& Is NOT A Forwarding Reference Here

This one bit me, so it deserves its own section.

```cpp
std::tuple<int, std::string> t;

auto&& [a, b] = t;                    // e binds to an LVALUE

std::forward<decltype(b)>(b);         // decltype(b) is std::string
                                      // -> std::string&&
                                      // -> silently MOVES FROM t
```

The familiar `std::forward<decltype(x)>(x)` idiom relies on `decltype(x)` giving
us back the *declared type* of the variable, which for a real forwarding
reference is either `T&` or `T`. That is what makes the idiom preserve value
category.

Structured bindings break this. `decltype` on a structured binding name gives
the **referenced type** of the element, not the declared type of a variable. So
`decltype(b)` is a plain `std::string`, `std::forward` happily produces an
rvalue reference, and we gut `t` even though the source was an lvalue and we
never wrote `std::move` anywhere. The compiler will not warn us.

Takeaways:

- Do not perfect-forward through structured bindings.
- If we truly must, C++23 gives us `std::forward_like<decltype(t)>(b)`, which
  forwards based on the value category of the *owner*, which is what we meant
  all along.
- Otherwise, forward the whole object and destructure at the far end instead.

While we are here, one more related surprise:

```cpp
auto&& [a, b] = std::move(x);   // moves NOTHING
```

The bindings are aliases, and `std::move` is just a cast. Nothing has been
moved yet. We still need `std::move(a)` at the actual point of use, and at that
point we should be very sure `x` is ours to move from.

## Summary

Write the qualifier that states our intent, using the same rules we would apply
to a plain `auto x = ...`. Nothing more exotic than that.

- `const auto&` - the most common one, for reading
- `auto&` - for mutating
- `auto` - for owning cheap values and temporaries
- `auto&&` - for generic code and proxy-yielding views, where it is not a
  stylistic hedge but the only thing that actually works

`auto&&` is not a smarter default. It is a specific tool for the case where the
value category is genuinely unknown to us. Reaching for it everywhere else
throws away the information we are trying to communicate to the next reader,
and, as the forwarding trap shows, occasionally does something we did not ask
for.

Until next time, keep learning and growing!

## References

- [Structured binding declaration](https://en.cppreference.com/w/cpp/language/structured_binding)
- [std::forward_like](https://en.cppreference.com/w/cpp/utility/forward_like)
- [std::views::enumerate](https://en.cppreference.com/w/cpp/ranges/enumerate_view)
