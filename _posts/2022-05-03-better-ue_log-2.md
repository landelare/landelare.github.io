---
title:  "A better UE_LOG, part 2"
excerpt: "Taking logging to the next level with new C++ features."
last_modified_at: 2023-05-11
---

**[←Part 1](/2022/04/28/better-ue_log.html)**,
**[Part 3→](/2023/03/21/better-ue_log-3.html)**

**Update: UE_LOGFMT fixes most of the issues with UE_LOG.
These techniques can still be useful if you want something more comfortable to use.**

---

In part 2, we'll revisit the logging system made in part 1 and look into how it
can be made better using new C++ features. UE4 can relatively easily be switched
to C++17, and as of UE4.27, C++20 is not that painful either (you'll need to ask
for `CppStandardVersion.Latest` and possibly disable shared PCHs).

UE5 is C++17 by default and C++20 is easy to turn on even with the binary Epic
builds: just add `CppStandard = CppStandardVersion.Cpp20;` to your Build.cs.
Your mileage may vary depending on what platforms you plan to support.

# C++17

## if constexpr

This is the big one. Remember that `typename = void` hack that we had to do in
part 1? What if we could write all of those as one single function so there is
no overloading hell?

`if constexpr`, like its name suggests, evaluates its condition at compile time.
However, as a bonus, you can call functions that don't exist if the compiler can
tell that the `if` is not taken.
Armed with this knowledge we can merge most `FillArgs()` overloads into one:

```c++
// This will get better in C++20
template<typename T, typename = int>
struct THasToString : std::false_type {};
template<typename T>
struct THasToString<T, decltype(std::declval<T>().ToString(), 0)> : std::true_type {};

// You still need this:
void FillArgs(FStringFormatOrderedArguments& Args)
{
}

template<typename F, typename... R> // no void!
void FillArgs(FStringFormatOrderedArguments& Args, F&& First, R&&... Rest)
{
    if constexpr (std::is_constructible_v<FStringFormatArg, F>)
        Args.Add(Forward<F>(First));
    else if constexpr (THasToString<F>::value)
        Args.Add(First.ToString());

    FillArgs(Args, Forward<R>(Rest)...);
}
```

Try writing the missing branches for `->GetName()` and `LexToString()`!
The solution is in Appendix A.

Did you notice the silent-but-horrible omission from the function above?
If `F` matches none of the conditions, the parameter will be skipped without a
warning. The obvious fix would be `else static_assert(false);` but that will
just make sure this will never compile.

The real fix is uglier: the condition has to depend on a template parameter to
make sure it only gets evaluated when the template is instantiated.

```c++
// The left-hand side doesn't matter; it will be false anyway.
else
    static_assert(std::is_void_v<F> && false);
```

A less hacky but slightly longer way of fixing this is to define a template
constant for `false`:
```c++
template<typename> constexpr bool bFalse = false;

// ...
else
    static_assert(bFalse<F>);
```

## Fold expressions

The `First, Rest...` parameter "unpacking" was a hard requirement in C++14, but
C++17 is able to iterate on parameter packs without resorting to recursion.

Running code on them is somewhat hacky and involves abusing `operator,` so you
might want to keep the recursive version anyway. Here it goes:

```c++
// No need for the empty overload if you do this!

template<typename... T>
void FillArgs(FStringFormatOrderedArguments& Args, T&&... Params)
{
    ([&](auto&& Arg)
    {
        // If you'd like a type shortcut:
        using F = decltype(Arg);

        // if constexpr goes here
    }(Forward<T>(Params)), ...);
}
```

This technique works with a separate single-param `FillArg` template function,
or alternatively you can name the lambda like
`auto FillArg = [&](auto&& Arg){/*code*/};`. Both of these are invoked this way:
`(FillArg(Forward<T>(Params)), ...);`

# C++20

## Template lambdas

The fold expression above can be rewritten using an explicit template parameter
instead of `auto&&` if you prefer:

```c++
// This:
[&](auto&& Arg)

// Becomes this:
[&]<typename F>(F&& Arg)
```

## Abbreviated function templates

Speaking of new template syntax, regular functions got parity with lambdas so
that you can use `auto` parameters:

```c++
// This is a template
void FillArgs(FStringFormatOrderedArguments& Args, auto&& First, auto&&... Rest)
{
    // ...
}
```

You don't have to commit to one style:
```c++
template<typename F>
void FillArgs(FStringFormatOrderedArguments& Args, F&& First, auto&&... Rest)
{
    // ...
}
```

## Concepts... almost

Concepts would've been great to replace the SFINAE hacks from part 1, but we've
already replaced them with `if constexpr` and don't need to do this anymore.
As a reference this is how it could've looked:

```c++
template<typename T>
concept THasToString = requires(T Value) { Value.ToString(); };

template<THasToString F, typename... R>
void FillArgs(FStringFormatOrderedArguments& Args, F&& First, R&&... Rest);

// or, with auto:
void FillArgs(FStringFormatOrderedArguments& Args, THasToString auto&& First,
              auto&&... Rest);

// or...
if constexpr (THasToString<F>) // no ::value, concepts work as constexpr bool
    Args.Add(First.ToString());
```

Not just full `concept`s, but `requires` expressions themselves also work as
`constexpr bool`. That means all the type checks can be directly contained
within `FillArgs()`:

```c++
// You still need this for the recursive version:
void FillArgs(FStringFormatOrderedArguments& Args)
{
}

template<typename F, typename... R>
void FillArgs(FStringFormatOrderedArguments& Args, F&& First, R&&... Rest)
{
 // if constexpr (std::is_constructible_v<FStringFormatArg, F>) still works
    if constexpr (requires { FStringFormatArg(Forward<F>(First)); })
        Args.Add(Forward<F>(First));
    else if constexpr (requires { First.ToString(); })
        Args.Add(First.ToString());
    else if constexpr (requires { First->GetName(); })
        Args.Add(First->GetName());
    else if constexpr (requires { LexToString(Forward<F>(First)); })
        Args.Add(LexToString(Forward<F>(First)));
    else
        // Pick a name you expect to never use for a requires-based solution,
        // or feel free to stick with one of the C++17 methods.
        static_assert(requires { First._spanish_inquisition_; });

    FillArgs(Args, Forward<R>(Rest)...);
}
```

It's not the nicest thing to read, but at least it's compact and self-contained.
Combine with fold expression lambdas to taste.

_If you're from the future and using C++23, you can use `static_assert(false)`
directly, without the `requires` hack._

## \_\_VA\_OPT\_\_

`##__VA_ARGS__` from part 1 was a nonstandard hack.<br/>
With C++20 there's finally a standard way to handle 0 parameters:

```c++
// Replace this:
#define MY_LOG(Format, ...) MyLog(TEXT(Format), ##__VA_ARGS__)

// With this:
#define MY_LOG(Format, ...) MyLog(TEXT(Format) __VA_OPT__(,) __VA_ARGS__)
```

On MSVC, this requires setting `bStrictPreprocessorConformance` to true in your
BuildConfiguration.xml or equivalent (`-StrictPreprocessor` on the command line).

## std::source_location

There's now a non-macro replacement of `__FILE__`, `__LINE__`, et al., it even
supports columns as standard!

Normally you'd use it like this and rely on the default value:

```c++
void MyLogMaybe(const FString& Message,
                std::source_location Location = std::source_location::current())
{
   // ...
}
```

But this doesn't really play well with variadic parameters. You can easily add
it to a macro wrapper though:

```c++
#define MY_LOG(Format, ...) MyLog(std::source_location::current(), \
                                  TEXT(Format) __VA_OPT__(,) __VA_ARGS__)
```

Note that `__func__` is also standard since C++11 if you cannot use C++20; it's
a variable though, not a macro. If you're reading this from the future, you
might find `std::stacktrace_entry` (C++23) useful, too.

<br/><br/><br/><br/><br/>

# Appendix: GetName/LexToString C++17 solutions

This is how you do `->GetName()`:

```c++
template<typename T, typename = int>
struct THasGetName : std::false_type {};
template<typename T>
struct THasGetName<T, decltype(std::declval<T>()->GetName(), 0)> : std::true_type {};
```

```c++
else if constexpr (THasGetName<F>::value)
    Args.Add(First->GetName());
```

Unfortunately `LexToString` is problematic. A lot of things attempt to take this
branch, for example pointers implicit converting to `bool`.
This will be a warning-as-error and not compile. I'd suggest keeping this one
last, making sure more complex cases are handled first and/or handling the types
causing warnings separately and explicitly (e.g. `std::is_pointer_v`).

```c++
template<typename T, typename = int>
struct THasLexToString : std::false_type {};
template<typename T>
struct THasLexToString<T, decltype(LexToString(std::declval<T>()), 0)> : std::true_type {};
```

```c++
else if constexpr (THasLexToString<F>::value)
    Args.Add(LexToString(Forward<F>(First)));
```
