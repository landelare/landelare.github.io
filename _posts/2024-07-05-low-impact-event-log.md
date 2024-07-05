---
title:  "Low impact event log"
excerpt: "A rarely-documented trick that can help track down heisenbugs."
---

Recently, I've been debugging an extremely rare multithreading bug, needing
thousands of re-runs of the same test case to even happen.
As per tradition, any attempts at trying to log what happened changed the timing
so much that the bug was gone.
I needed something with minimal impact on timing.

There is a technique that can be used in this situation, although I don't know
its name or if it even has one at all.
"Append buffer" might be the closest match; something very similar is done in
shader programming, although not for this purpose.
Various low-impact trace facilities, such as
[ETW](https://learn.microsoft.com/en-us/windows/win32/etw/about-event-tracing),
are also designed to reduce their runtime impact using similar methods.

There are two parts to this: an array to hold events, and an atomic index or
pointer into that array.

To make things somewhat readable, the array can be based on an enum (this was
the real enum used when debugging the problem mentioned earlier):

```c++
enum class Event
{
    None,
    CoroScopeExit,
    CoroPreTrigger,
    CoroPostTrigger,
    CoroFinalAwait,
    TestPreWait,
    TestPreCancel,
    TestPostCancel,
    TestPostTrigger,
    TestPostPump,
} EventLog[10]{};
```

In the spirit of debugging hacks, this code defines the enum, creates an array
of 10 out of it, and zeroes it out in one fell swoop.
This is why `None` is important to have as the first entry.
The array should be sized to comfortably accommodate the number of entries
expected; in this case, it was around 7.

This array comes before the test case so that it's alive throughout its entire
execution.

We also need to know where to write the next event in the array.
In my case, this line was right under `EventLog`:
```c++
std::atomic<int> Idx = 0;
```

Adding an entry to the log is done by simply writing to the array at `Idx++`:

```c++
EventLog[Idx++] = Event::CoroPreTrigger;
CoroToTest->Trigger();
EventLog[Idx++] = Event::CoroPostTrigger;
```

`operator++` on `atomic<int>` is special: it will do an atomic increment in one
step instead of the problematic read-modify-write that can happen with a normal
`int`.

<sup>
Perhaps it's worth mentioning that C++'s `volatile` is often misused for this.
Its **only** purpose is to indicate to the compiler that the act of reading
from or writing to such a variable is important.
Its multithreading benefits are part luck, part compiler extensions, especially
with MSVC on x64, where it does behave like an atomic by default.
This can lead to surprise bugs when porting code to Arm, where it does not.
</sup>

After this, all that's left is to detect and breakpoint the bad case, and the
order of events will be readily available in `EventLog` to be read with your
debugger... usually.

I got lucky because the bug was reproducible in a debug build, where writes to
the array tend to happen reliably.
Of course, we can't forget about everyone's favorite release-only bugs.
If writes to the array turn out to not arrive reliably, try changing it to be an
array of atomics, or indeed, `volatile`.

If you cannot reliably breakpoint the failure case (especially in release
builds), another trick that you can use is to compile the breakpoint directly
into the executable.
All major compilers have such an intrinsic:
```c++
if (/*stuff failed*/)
    __debugbreak();
```
On some platforms, it's called `__builtin_trap()`.

An alternative usage of the same idea is to `|=` bit flags into an atomic int
if you only want to know what events have happened, not necessarily their order.
