---
title:  "Should I follow the Epic coding standard?"
excerpt: "Things that I wish I knew when I started and had to learn the hard way.
This one will be controversial!"
last_modified_at: 2024-01-03
---

This one will be controversial but I really wish someone told me all of this.
When I initially started working with Unreal Engine I defaulted to following
the official standards fully, not having known better.
This remains excellent advice if you don't but I do now, and maybe this will
help inform your choices to not end up with some of the same regrets and
technical debt that I now have to deal with.

There are a few situations where the answer is an obvious "yes":
* You're working on engine code intending to submit it to Epic
* You're making a plugin for the Marketplace (at least on your public API)

There's also clear value in having a common coding convention across multiple UE
teams and different companies so deviating from it too far would also be silly.
"Modern C++" still retains some controversy and Unreal Engine, like the majority
of C++ projects disables exceptions and RTTI which neuters some of it when it
comes to exception safety.
UnrealHeaderTool also imposes some of its own limitations, most notably around
namespaces.

With all that said, there is also a case to be made for **not** following these
conventions fully.
The main reason behind many of them is to be consistent with the existing heaps
of legacy engine code, which doesn't apply to your projects.
Epic themselves are getting more lax when it comes to following their own
conventions (seems to be different per module) and it was officially relaxed a
little for UE5.

What this is not is some kind of "better" or alternative coding standard, or
your defense if you want to do stuff like `for (auto i{0};...`. Please don't, but
take some time to determine if completely following the Epic standards is really
the best for your team, or if deviating from it would be better. In our case
there was a gradual slide away from the Epic standards which has led directly to
bugfixes and cleaner code.

Not all of these items are from Epic's
[official coding standard](https://docs.unrealengine.com/5.3/en-US/epic-cplusplus-coding-standard-for-unreal-engine/).
Some are widespread and perhaps misguided/outdated practices that I've seen
within the community that I'll address without distinguishing between the two.

## PascalCase everything

Epic seriously overuses PascalCase and doesn't use camelCase at all... at least
as far as the coding standard documentation goes. You can find multiple
instances of them using it anyway, and there's clear value in not leaving
camelCase on the table.

Consider using it for some of: function parameters, locals, class fields, maybe
only for non-properties or private, etc.
Up to you where you split it.
Local variables are probably the strongest contenders for it.

## Pointers for UObjects

Reading engine code I soon got the impression that every `UObject` (including
actors of course) has to be stored and passed around as a pointer.\*
Epic code often uses `UObject&`, and `TArray::Sort` even enforces it (but don't
use that one in particular, see below on `std::sort`).

\*Let's ignore `TObjectPtr<T>` which might or might not go away.
As of writing it's optional and its existence can be largely ignored.

## int32

You're supposed to use `int32` for every `int` that may be serialized, e.g.,
on UPROPERTYs.
This is correct to do, but also practically speaking you don't need to do it.
`int` is widely expected to remain exactly 32 bit on all platforms that run UE,
forever.

If `int32` is not used everywhere by default it stands out as explicitly marking
something a 32-bit value, bringing extra attention to this which can help guide
your reader.

## auto

Epic has a very restrictive stance on `auto` that goes directly against
established modern C++ practices, mainly driven by a misunderstanding of its
motivation as a typing reduction tool instead of the correctness, performance,
maintainability, and robustness improvement that it mainly provides.
[1](https://herbsutter.com/2013/06/07/gotw-92-solution-auto-variables-part-1/)
[2](https://herbsutter.com/2013/06/13/gotw-93-solution-auto-variables-part-2/)
[3](https://herbsutter.com/2013/08/12/gotw-94-solution-aaa-style-almost-always-auto/)

Even if you disagree with one of the foremost experts on C++, there are
situations where you can use it with no drawbacks (imagined or real).
It's important to distinguish between "decorated" and regular `auto`, see below.

I can personally attest to this: porting an `auto`-heavy UE4 codebase to UE5
highlighted and prevented a lot of implicit float-double conversions
(performance) that would have had issues when converting large LWC coordinates
beyond the old WORLD_MAX (correctness, maintainability), something that Epic has
been manually tracking down and fixing for multiple 5.x releases.

`auto` is completely harmless when the type is obvious, and even on the same line:

```c++
auto* thing = Cast<UWhatCouldThisPossiblyBe>(object);
```

You can also consider using it when the type is not immediately visible, but
it's still obvious what the variable will be:

```c++
auto& LAM = GetWorld()->GetLatentActionManager();
```

Note that I was mainly using `auto*` and `auto&`, and that's what I'd suggest
using most of the time in code interacting with the engine, where UObjects don't
conform to RAII, and interfaces are not ~~consistently~~ designed, and
TObjectPtr's existence makes things even more complicated.
<br><sub>
Exercise: guess what `TMap::Find`, `TMap::FindChecked`, and `TMap::FindRef` return.
Did you get it right?
</sub>

```c++
const auto a = Cast<AActor>(Object); // This is not const AActor*!
auto b = GetSomething(); // Is this a copy? A reference?
auto&& c = GetSomething(); // This can be both an lvalue or an rvalue reference!
```

There's also an auto-like syntax for returning a default value, `return {};`
that is less chatty.
Otherwise `{}` is best limited to `std::initializer_list` overloads: even the
STL itself is defined in many cases to construct values with `T(params)` instead
of `T{params}`.

## float literals

During UE4->UE5 LWC porting a lot of `0.0f`s and `1.0f`s needed to be fixed.
`0` and `1` worked for both `float` and `double` without ambiguous overload
errors and it's what I would've used in hindsight, especially if it's generic
code that still needs to support both types.
Obviously it doesn't apply if you need to select a particular overload.
Chances are you can also settle on `double`s forever and use `0.0`/`1.0`, but
there are still `float` to `double` changes happening after UE5.0.

## Structured bindings

Epic employees are not allowed to use this, but you can! It makes code a lot
cleaner when you don't have to create your function's return values at every
call site just to give to a `T&` out parameter.

If you need to reassign values to existing variables, there's `std::tie` for
`std::tuple` and `Tie` for `TTuple` (see below on STL usage).

```c++
auto [two, returns] = YayNoOutRefs();
std::tie(two, returns) = YayNoOutRefs();
Tie(two, returns) = YayNoOutRefs();
```

## Use of namespace std

It's hard to give a short, easy-to-remember guideline for this one.
There are plenty of issues with the STL, but Epic's replacements have their own
issues as well. When to use which is mostly situational and based on experience
and even engine version.
The official conventions now state:

> When there is a choice between a standard library feature instead of our own,
> **prefer the option which gives superior results**, but bear in mind that
> consistency is also valued highly.

You don't have to be consistent with Epic's decades' worth of code in your own
project! :)

There was a strong argument to be made for not using the STL at all simply for
consistency's sake, but Epic themselves have started using the STL in some
situations (see below), this is unfortunately (or fortunately?) getting weaker
and weaker as time goes on: future Unreal code will look like a mixture between
Epic and STL style anyway.

### Containers

Containers are one of the weakest points of the STL, and you're forced to use
the TContainers for anything exposed to the reflection system anyway.
The Unreal versions generally look and perform better, have a better syntax for
custom allocators, but are sloppier when it comes to type safety.

For instance, if you try and `push_back` an immovable type into a `std::vector`,
it will correctly not compile and prevent any bugs that might arise from this.
`TArray` will happily memcpy you around without a warning.
Keep this in mind when you're working with types that have special or unknown
needs (such as in template code).
You might want to add a few `static_assert`s in this case.

### Algo

On the other hand, `<algorithm>` is one of the best parts of the STL, and Epic's
`Algo` namespace has some questionable implementation choices and is often
outperformed by STL algorithms (for instance `std::sort`).
At this point I default to STL algorithms, deferring to Algo only in situations
where the STL-like iterators that the UE containers provide are not enough.

The [EASTL](https://github.com/electronicarts/EASTL) might work even better if
you're OK with adding the extra dependency.

`std::ranges` improves the begin/end syntax of regular `<algorithm>` (where
`Algo` remains better) in C++20 but it's more or less incompatible with UE
containers.

You can get a lot of mileage out of this small adapter function:
```c++
#include <span>

auto AsSpan(auto& Container)
{
    return std::span(GetData(Container), GetNum(Container));
}
```

### Smart pointers

`TUniquePtr` has some bizarre bugs in UE4 that were fixed in UE5, and is mostly
a "just because" replacement that serves no additional function.

`TSharedPtr` performs similarly to `std::shared_ptr` when in its default,
thread-safe mode. For game thread stuff, making it not thread safe significantly
improves its performance (also applies to `TWeakPtr`/`std::weak_ptr`).

`TSharedFromThis` has some weird limitations compared to
`std::enable_shared_from_this`.
If you're hitting mysterious assertions, consider using the STL version.

### std::move / std::forward

`MoveTemp` ~~is~~was a great replacement for `std::move` because it enforces
move semantics with a `static_assert`.
On the other hand `MoveTempIfPossible` is exactly like `std::move`.

`std::forward` and `Forward` are equivalent.

For all of these, the STL versions use `static_cast` while the UE ones use
C-style casts.

_Update: MSVC in VS2022 17.5 has
[added intrinsics](https://devblogs.microsoft.com/cppblog/improving-the-state-of-debug-performance-in-c)
for std::move and std::forward that greatly improve performance in debug builds,
making this choice less obvious._

### Miscellaneous

`TAtomic` is an easy case. `std::atomic` is the **officially**-preferred atomic
and should always be used instead.

The Epic type traits and SFINAE helpers (`TEnableIf`, etc.) are also officially
deprecated in favor of their STL counterparts. If you're using UE5, C++17 can
help eliminate a lot of SFINAE with `if constexpr` and C++20 lets you go even
further beyond with `concept`s if you can opt in.

The same applies to `std::numeric_limits`, etc. If an Epic reimplementation
provides no benefit but it's only there for the sake of it, it's worth
defaulting to the STL which receives more testing and focused dev attention.

`TTuple` and `TVariant` are incomplete reimplementations of their STL variants.
They got better in UE5 but there are still STL-exclusive features (like
a working `operator==`...) and they have started becoming STL hybrids themselves
due to language features being defined in terms of `std::tuple_size`,
(lower-case) `get<i>()`, etc.
~~Since there's no BP or reflection support for these it's worth considering not
using them at all.~~<br>
_UE5.1 update: These now have FArchive support, which makes the decision less
straightforward._

`TFunction` has some bugs (as of 5.0.2) related to perfect forwarding and who
knows where else. I stopped looking when my crashes got fixed by nothing but
changing it to `std::function` which is an all-around better replacement.

## const locals and parameters

This is not really an Epic thing, but I still see it often, usually because some
IDE or productivity tool recommends that you _can_ do this.
There's an academic/theoretical benefit to this: it can prevent accidentally
overwriting a local when it was not intended. In practice, this is not really an
issue - C# doesn't have a similar language feature and the bugs that are
supposedly not prevented by it mostly don't exist.

Overusing it adds noise and takes attention away from the real, important
`const`s, as well as directly impacting performance: moving needs `T&&`, not
`const T&&`, and many `const` method overloads return copies instead of
references.

Of course, this doesn't apply to pointers or references to `const` things.

## Language or engine features

### NULL
For some reason, `NULL` lives on in some questionable tutorials.
You should always use `nullptr` instead.
`NULL` is the integer constant `0` in C++ and its only purpose is to keep
ancient code on life support.

### typedef
`typedef` is considered legacy, the new syntax is `using MyAlias = int;`.
It gets even better for function pointers and it can also be a template!

### mutable and FORCEINLINE
Engine code seriously overuses these, they should be very rare.
Even if you're already at the point where you're genuinely microoptimizing
calls, you can outperform `FORCEINLINE` by moving some parts of a function to
a wrapper caller macro.

Other, game-focused libraries, such as the EASTL, use these far less often than
Epic.

### Legacy ... varargs
These are crash-prone, not type-safe, and all-around horrible.
They don't even know what `float` is. Use variadic templates instead.
There's a subtly-horrible syntactic difference between them, make sure you place
the `...` in the right place:

```c++
void Crashf(T x, ...); // C
void Crashf(T x...); // still C!
void Safef(T... x); // variadic template
```

### GENERATED\_...\_BODY
`GENERATED_UCLASS_BODY`, `GENERATED_USTRUCT_BODY`, etc. are all obsolete.
It's always `GENERATED_BODY()` for everything. Your code might break if it
relied on the legacy macros declaring its constructor, simply declare it to fix
it.
