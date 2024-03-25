---
title:  "A better UE_LOG"
excerpt: "Step 1 of a journey towards hopefully better logging in Unreal Engine."
last_modified_at: 2024-03-25
---

**[Part 2→](/2022/05/03/better-ue_log-2.html)**

**Update: UE_LOGFMT fixes most of the issues with UE_LOG.
These techniques can still be useful if you want something more comfortable to use.**

I have also published a [sample library](https://github.com/landelare/llog) that
implements some of the ideas presented by this series.

---

This post assumes that you're already familiar with the various UE logging
facilities such as `UE_LOG` or `GEngine->AddOnScreenDebugMessage` and examines
a few approaches on how we can make something better.

Both of the engine methods have their respective drawbacks:
`UE_LOG` needs 2 extra parameters even if you Just Want To Debug Print™, it uses
the horrible legacy `...` parameter passing and `printf`-style format strings
while the other requires you to format your string yourself.
`FString::Printf` suffers from the same issues: you need to use `TEXT()` and the
list goes on.
Just give me a log that does its job and stays out of the way!

This is deliberately not being made available as a plugin.
Every project has slightly different needs for logs: you might prefer format
strings in `printf`/`std::format`/`FString::Format` flavor,
`VariadicStyle("X=",x," Y=",y)`, `operator<<` or simply hate on-screen debug
messages.
You might want to have a `LOCTEXT_NAMESPACE`-like macro to mark where your logs
are coming from, others might find that a waste of time.

Since there's no best logging and everybody would need to heavily customize a
plugin, instead we'll go through the ideas and techniques that will let _you_
come up with logging that suits _your_ project for that nice warm feeling of
Pride and Accomplishment™, but with plenty of copy-pastable code blocks so you
don't need to work _too_ hard. ;)
(plus making this into a plugin is about as much boilerplate as the code itself)

Part 1 will get to a basic solution strictly within the realm of UE4 and C++14,
future parts will show how new C++ features can be used to improve/remove the
hacks imposed by these limitations.

## Avoiding C varargs `(...)`

First things first, we want to get rid of C and have something that's type safe.
Having compile-time visibility into types will enable a lot of extra
functionality later on as well as prevent bugs (try printing an `int` with `%s`
and watch it crash).

[Variadic templates](https://en.cppreference.com/w/cpp/language/parameter_pack)
are the perfect fit because you retain all type information:

```c++
// This is what we're trying to avoid:
// FString::PrintfImpl(const TCHAR* Fmt, ...);

template<typename... T>
void MyLog(const TCHAR* Format, T&&... Args)
{
    // ...
}
```

`T&&` parameters enable **perfect forwarding**, you can read more about it
[here](https://www.modernescpp.com/index.php/perfect-forwarding) or
[here](https://en.cppreference.com/w/cpp/language/reference#Forwarding_references).
Taking parameters this way not just ensures that you can take temporaries in
your logging function (like `MyLog(TEXT("..."), 1, FString())`) but it can also
help avoid needless copies when we'll process these parameters later.

`FString::Format` is the most unusual and hardest to implement of all the
options presented so this approach will be used throughout this article:

```c++
// Usage: MY_LOG("Hello {0}", 1);
#define MY_LOG(Format, ...) MyLog(TEXT(Format), ##__VA_ARGS__)
template<typename... T>
void MyLog(const TCHAR* Format, T&&... Args)
{
    FStringFormatOrderedArguments OrderedArgs;
    FillArgs(OrderedArgs, Forward<T>(Args)...); // We'll write this next
    FString Message = FString::Format(Format, MoveTemp(OrderedArgs));

    GEngine->AddOnScreenDebugMessage(-1, 5, FColor::Cyan, Message);
    UE_LOG(LogTemp, Display, TEXT("%s"), *Message);
}
```

You can scroll down to the bottom of the article for other starters and build
from there instead if you prefer another approach.
Go wild with macro wrappers, put this into a class, etc., whatever fits your
preferences and project the best.
The original project where this code was first written is using a combination of
a class and some macros.

## Converting arguments

Since we have compile-time type information for arguments, we can use this to
provide some overloads for `FillArgs` and give it a little more intelligence
than just printing raw pointer values or relying on the user to call methods on
them like `->GetName()`.

_Nothing_ needs to be implemented first.
This looks useless but it's very important, this is where the recursion will
terminate:

```c++
void FillArgs(FStringFormatOrderedArguments&)
{
}
```

`FStringFormatOrderedArguments` already has a bunch of Add overloads so let's
use them if they're available. We'll need to separate the `T&&...` arguments, do
some SFINAE on the first, and pass on the rest to the same function. This is why
the empty overload above is important.

```c++
template<typename F, typename... R>
auto FillArgs(FStringFormatOrderedArguments& Args, F&& First, R&&... Rest)
    -> std::enable_if_t<std::is_constructible_v<FStringFormatArg, F>>
{
    Args.Add(Forward<F>(First));
    FillArgs(Args, Forward<R>(Rest)...);
}
```

For the purists, it might be news that Epic updated the UE coding conventions
and  `<type_traits>` is now
[fair game](https://dev.epicgames.com/documentation/en-us/unreal-engine/epic-cplusplus-coding-standard-for-unreal-engine#useofstandardlibraries).
The UE versions of these are stuck on C++11 so I'd recommend moving on and using 
the shorter/better STL versions, as weirdly as it might sound.

That's it, check your shiny new log out! Just this code will already work for
every number and string. Don't forget to put the implementation of `FillArgs`
above `MyLog` so that the compiler can see it.

## Extending `FillArgs`

That's great! We can just SFINAE different overloads, ensuring that for every
type only one `FillArgs` is visible, and extend this as much as we want...

...is exactly what I thought, too. I probably wouldn't be writing this article
if that worked.
If you try doing that, you'll get arcane errors complaining about ambiguous
overloads. There are some C++ language rules at play that aren't worth covering
here, but long story short for every overload you'll need to add an extra
useless template parameter to make them "different enough" for SFINAE:

```c++
// Arguments that have .ToString()
template<typename F, typename = void, typename... R> // +1 extra void
auto FillArgs(FStringFormatOrderedArguments& Args, F&& First, R&&... Rest)
    -> std::void_t<decltype(First.ToString())>
{
    Args.Add(First.ToString());
    FillArgs(Args, Forward<R>(Rest)...);
}
```

Let's keep going:

```c++
// Arguments that have ->GetName()
template<typename F, typename = void, typename = void, typename... R> // +2 voids
auto FillArgs(FStringFormatOrderedArguments& Args, F&& First, R&&... Rest)
    -> std::void_t<decltype(First->GetName())>
{
    Args.Add(First != nullptr ? First->GetName() : TEXT("null"));
    FillArgs(Args, Forward<R>(Rest)...);
}
```

Note that `std::void_t` is used for these for "does this member exist?" SFINAE.
`std::enable_if_t` works for `bool` conditions.

Stop here and try implementing your own overload for `LexToString()` as a
challenge. The solution is in Appendix B if you're stuck.

## Wrapping things up

That's pretty much it for part 1!
You should have a solid baseline for implementing your own, less painful debug
prints.

See you in [part 2](/2022/05/03/better-ue_log-2.html) where we'll look at how
to untangle this `template` mess with new language features added in C++17 and
C++20!

<br/><br/><br/><br/><br/>

## Appendix A: Alternative starters

As promised, here are some alternative starters in other styles.
You may want to consider wrapping these in macros so that you don't need to add
`TEXT()` everywhere, you can add colors for log levels, etc.

If you want to keep using `FString::Printf` no matter what you can simply pass
your params through and still benefit from a shorter call.
This works for `std::format`, too (in C++20), but at least that's type safe.
You'll have some practical issues with `TCHAR` vs `char` that you can work
around by passing every parameter through a non-recursive `FormatArg()` that's
written like `FillArgs` above. Return your `T&&` parameter formatted as
`std::string`. I would not recommend attempting that for `Printf` and `...`
varargs.

```c++
// Usage: MyLog(TEXT("Hello %d"), 1);
// For FString::Printf specifically you can't take a TCHAR*
template<std::size_t N, typename... T>
void MyLog(const TCHAR(&Format)[N], T&&... Args)
{
    FString Message = FString::Printf(Format, Forward<T>(Args)...);
    // UE_LOG etc.
}
```

If you want to just format and append strings without an explicit format string:

```c++
template<typename... T>
void MyLog(T&&... Args) // Usage: MyLog("Hello ", 1);
{
    FString Message;
    // Same overloading technique as the main approach:
    AppendToMessage(Message, Forward<T>(Args)...);
    // UE_LOG, etc.
}
```

For the `operator<<` fans:

```c++
struct FMyLog // Usage: FMyLog() << "Hello" << i;
{
    FString Message;

    // Write a constructor if desired to take a color, etc.

    template<typename T>
    FMyLog& operator<<(T&& Arg)
    {
        // Same overloading technique, but without recursion:
        Message += FormatArg(Forward<T>(Arg));
        return *this;
    }

    ~FMyLog()
    {
        // UE_LOG, etc.
    }
};
```

Alternatively, you can combine a format string and `<<`:

```c++
struct FMyLog // Usage: FMyLog(TEXT("Hello {0}")) << i;
{
    const TCHAR* Format;
    FStringFormatOrderedArguments Args;

    FMyLog(const TCHAR* Format)
        : Format(Format)
    {
    }

    template<typename T>
    FMyLog& operator<<(T&& Arg)
    {
        // You guessed it: same technique, no recursion
        FillArg(Args, Forward<T>(Arg));
        return *this;
    }

    ~FMyLog()
    {
        FString Message = FString::Format(Format, MoveTemp(Args));
        // UE_LOG, etc.
    }
};
```

If you want to use `std::format` with the Microsoft STL, as of writing you'll
need to apply a hack to expose a few things to `namespace std` that
[should be there already](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p2508r1.html).

```c++
#if defined(_MSVC_STL_VERSION) && __cpp_lib_format <= 202110L
namespace std
{
    template<typename CharT, typename... Args>
    using basic_format_string = _Basic_format_string<CharT, Args...>;

    template<typename... Args>
    using format_string = _Fmt_string<Args...>;

    template<typename... Args>
    using wformat_string = _Fmt_wstring<Args...>;
}
#endif
```

## Appendix B: `LexToString` solution

```c++
template<typename F, typename = void, typename = void, typename = void, // +3
         typename... R>
auto FillArgs(FStringFormatOrderedArguments& Args, F&& First, R&&... Rest)
    -> std::void_t<decltype(LexToString(Forward<F>(First)))>
{
    Args.Add(LexToString(Forward<F>(First)));
    FillArgs(Args, Forward<R>(Rest)...);
}
```
