---
title: "Weak scope exit"
excerpt: "A small, but useful feature currently missing from Unreal."
---

`ON_SCOPE_EXIT` is great for pretending that C++ has a `finally` or `defer`
keyword, and cleaning up things that don't support RAII (looking at you,
UObjects).<br>
<sup>
In the _far_ future, there will be a std::scope_exit, but it's not even close to
standardization right now.
</sup>

# Usage

In case you're not familiar, it's used like so:
```c++
void ProcessAnything()
{
    Something->Enable();
    ON_SCOPE_EXIT
    {
        Something->Disable();
    };
    ProcessEnabledSomething(Something);
}   // Something gets disabled here
```

Its implementation is nothing more than the usual local variable with a
destructor, its main benefit is that it's already there in the engine.

In case you're working with exceptions enabled, this is even exception-safe and
will **always** run, even if your stack is getting unwound due to an exception.

# Issues

`ON_SCOPE_EXIT` **always** running is its main strength, but also one of its
greatest weaknesses.<br>
What if you don't want to run if `this` is a UObject that's not valid anymore?

```c++
FWeakObjectPtr WeakThis(this);
ON_SCOPE_EXIT
{
    if (!WeakThis.IsValid())
        return;
    // Do the thing
};
```

Although this is relatively rare to encounter in synchronous game-thread-based
code, it needs to be considered if you're, e.g., multithreading, using callbacks,
or <!--♪-->coroutines<!--♫-->.

Doing this every single time will get old quick.

Not getting this right can also result in very sneaky, hard-to-track-down bugs,
since you'll need to get it wrong at _just the right moment_ for the GC to cause
you to crash, and even that is not 100%.
Usually it will just deal with data it shouldn't be dealing with, possibly
leading to silent corruption.

# Implementation

With just a little tweak, you can make a weak version of `ON_SCOPE_EXIT`.
The following code block is usable as is in a .h file.

I'm using the `+=` operator instead of the engine's `+`, because it has much
lower precedence.<br>
`+` is a weird choice.
If Epic wanted strong precedence, `*` would've been stronger.

```c++
#define ON_SCOPE_EXIT_WEAK auto PREPROCESSOR_JOIN(_weakScopeExit_, __LINE__) = \
                           ::WeakScopeExit::FHelper(this) += [&]
namespace WeakScopeExit
{
template<typename F>
struct TWeakScopeExit
{
    FWeakObjectPtr Weak;
    F Finally;
    explicit TWeakScopeExit(const UObject* Object, F&& Callback)
        : Weak(Object), Finally(MoveTemp(Callback)) { }
    ~TWeakScopeExit() { if (Weak.IsValid()) Finally(); }
};

struct FHelper
{
    const UObject* Object;
    explicit FHelper(const UObject* Object) : Object(Object) { }
    template<typename F>
    TWeakScopeExit<F> operator+=(F Callback)
    {
        return TWeakScopeExit(Object, MoveTemp(Callback));
    }
};
}
```

This will only run if `this` is valid, which is great to use from lambda
callbacks in a member function, or directly in member <!--♪-->coroutines<!--♫-->.

# Other variants

* It's trivial to modify the macro to work with any other object instead:
```c++
#define ON_SCOPE_EXIT_WEAK_OBJ(Obj) \
    auto PREPROCESSOR_JOIN(_weakScopeExit_, __LINE__) = \
    ::WeakScopeExit::FHelper(Obj) += [&]
```

* If you wrap the entire code block in `#ifndef ON_SCOPE_EXIT_WEAK`/`#endif`, you
can automatically let any future engine version "win" if this feature gets added.
Of course, you can also use a custom name for the macro to prevent collisions.

* If you are in an environment with exceptions enabled, you can easily modify
this to only run if there is/isn't currently an exception being thrown.
`std::uncaught_exceptions()`, when called from `~TWeakScopeExit()`, will return
a nonzero number in case there's currently an exception being processed.
For UE4, you'll want the deprecated `std::uncaught_exception()` without the `s`.
