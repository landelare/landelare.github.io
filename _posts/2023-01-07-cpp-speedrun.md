---
title: "Unreal C++ speedrun"
excerpt: "Gain months' worth of Unreal C++ experience in a single article."
last_modified_at: 2024-10-24
---

This article assumes significant experience with C++, but not necessarily within
the Unreal Engine ecosystem.
Some C# is also required, which is used for Unreal tooling, such as the build
system and code generators.

Although most of your knowledge will carry right into Unreal, there's a lot that
I had to find out the hard way.
Some things are either not documented at all, or only "documented" in the form
of unstructured multi-hour Twitch livestreams archived on YouTube.

That said,
[this](https://dev.epicgames.com/community/learning/courses/KJ/converting-blueprint-to-c/)
series of videos from the official learning site is relatively good, even if the
title of this course is somewhat misleading.

# Language choice

Blueprints or C++? The only correct answer is **both**.
BP is not limited to the spaghetti node graph but also provides essential
features such as providing references to assets, configuration, or runtime types
that don't exist in C++.

Don't get frustrated looking for C++ resources and finding that most of the
documentation or online resources target BP.
Most of those nodes are wrappers of some C++ engine function that you were
really looking for, and are often found by searching for `ExampleName` or
`"Example Name"` in the engine source code.

It's very common to build a C++ base class and a BP subclass together that
form a single logical unit, especially with actor classes.
C++ is great at core architecture, performance, and being source controlled;
BP is great at asset references, visuals, ease of programming for non-technical
people, and fast iteration.

Since most blueprints are .uassets and essentially unmergeable, you'll need to
consider this for your version control setup.
The most common approach is to use exclusive checkouts and file locks, although
good team communication scales surprisingly well.
Some .uassets can be merged (e.g., data assets in 5.3, data-only BPs in 5.4),
but these are the exceptions and they can't be matched by a filename pattern in
.gitattributes or a P4 typemap.

There is a thing as too much C++.
Do not ever hardcode an asset path into C++, these often lead to broken
references and packaging errors.
This includes a total ban on `ConstructorHelpers` and similar functionality.
Configuration is also better done using the engine's functionality instead of
piles of #defines or constexpr values in headers.

There's an official
[coding standard](https://dev.epicgames.com/documentation/en-us/unreal-engine/epic-cplusplus-coding-standard-for-unreal-engine).
It's relatively unusual in that it overuses PascalCase to the extreme.
Some studios follow it, some don't; both approaches have merit.
Not even the engine follows it consistently.
I wrote a [separate article](/2022/06/23/epic-conventions.html) on this topic.

# Development environment

Windows is the primary (arguably only) platform fully supported for development.
Linux and macOS do not have Epic's full attention for the Editor, but support is
better for them as target platforms for shipped games.

There are two practical choices for an IDE, Visual Studio and JetBrains Rider.
As of writing, Rider on Windows requires you to have a license for Visual Studio
according to JetBrains (which may be Community if you're eligible).
For the other two platforms, you're left with only Rider.
Visual Studio **Code** is an entirely different product, which is widely
considered broken for Unreal development.
CLion, despite being the C\+\+ IDE in the IntelliJ world, does not support
Unreal.

Old versions of Visual Studio and Rider (from years ago) did not properly
support Unreal Engine projects.
Make sure your IDE is up to date.

The decision between Visual Studio and Rider mostly boils down to price and GUI
preference.
If your computer has 16 GB of RAM or less, you might also want to consider that
VS generally uses less RAM than Rider.

VS Community is completely free, even for commercial use, assuming that you're
not working for an enterprise.
The full terms are
[here](https://visualstudio.microsoft.com/license-terms/vs2022-ga-community/);
what counts as an enterprise is described near the bottom of the first page.<br>
Rider is free for non-commercial use only.

If you want to pay, essentially all of Rider's features are available as a VS
add-in in the form of ReSharper for slightly cheaper.

If you're not eligible for Community, having a VS license needs to be factored
into Rider's price on Windows.<br>
There's also a discounted ReSharper+Rider bundle.
Some people like using both at the same time, generally preferring Rider for
editing code and VS for debugging in this case.

Although JetBrains is heavily pushing for subscriptions, perpetual licenses are
still available in a
[hybrid scheme](https://sales.jetbrains.com/hc/articles/207240845).

## VS2022
{:.no_toc}

VS2022's Unreal support is currently in preview, and it has to be explicitly
installed in the Visual Studio Installer.
It's not stable yet, but it improves the experience tremendously on VS2022
17.11+ and Unreal 5.4+ compared to the traditional way of using an
Unreal-generated "fake" .sln file.

I'd suggest trying .uproject first (File -> Open -> Unreal Engine project),
falling back to a generated .sln only if things break, or if your project is too
old to be eligible.
It takes several minutes for the engine code to get parsed at first; things will
appear broken until this is complete.
This is normal for large C++ codebases.

Using the .sln, VS has a harder time than with .uproject: IntelliSense often
breaks, displaying nonsensical errors and polluting the Error List with phantom
entries.

The Error List is essentially useless for Unreal solutions; you'll always want
to look at the Output panel for the full, unabridged errors.
[Here](https://dev.epicgames.com/documentation/en-us/unreal-engine/setting-up-visual-studio-development-environment-for-cplusplus-projects-in-unreal-engine#turnofftheerrorlistwindow)'s
how to turn it off and prevent it from coming back.<br>
**If an error does not appear in Output, it's not an error.**

IntelliSense sometimes "gives up" on some files entirely with a .sln:
this usually manifests in the files losing highlighting, or getting red
squigglies on everything.

To fix this and to get a better experience with .sln, there are two popular
commercial extensions that add full Unreal support and various other convenience
features:

* [ReSharper](https://www.jetbrains.com/resharper-cpp/) (JetBrains)
* [Visual Assist](https://www.wholetomato.com) (Whole Tomato)

Visual Assist's quality has been declining for a while, and its development lags
behind ReSharper.
Its users also report the purchasing/renewal process as being annoying.
As such, I cannot recommend it, but if you're already using it, you know what to
expect.

UE ships with a VS extension called UnrealVS (found in Engine/Extras/UnrealVS),
which is highly recommended.
Even for non-Unreal projects, it provides a convenient way to set command-line
arguments for debugging.
The third-party
[Unreal Wizard](https://marketplace.visualstudio.com/items?itemName=Fiquegnima-productions.UnrealWiz)
extension is also strongly recommended for additional/replacement functionality
where UnrealVS is broken.

Since the .sln that Unreal generates doesn't have real folders in it (only
"filters"), attempting to create new files using built-in methods will cause
them to default to somewhere in Intermediate where they won't build.
Use one of the extensions above to create new files, or create them "raw" in the
filesystem and regenerate your solution to pick them up.
This is not an issue with .uproject, which provides real folders.

If you're using VS with a .sln, but without Visual Assist or ReSharper, I highly
recommend having Unreal Wizard's toolbar on your main toolbar for its "Generate
Project Files" button.
When IntelliSense breaks, pressing that and reloading often resolves the issue.
If that doesn't work, the next step is deleting the `.vs` folder in your project.

Check if you have accidentally installed IncrediBuild from the VS installer.
If you don't have a license and a server farm set up for it, it will only slow
your builds down, so get rid of it.
It's also been reported to break shader compilation in this case.

## Rider
{:.no_toc}

Since Rider contains ReSharper out of the box, it mostly Just Works™.
Although it can work with the Unreal-generated .slns, too, its .uproject support
is mature and recommended to use.

The Error List advice from VS partially applies to Rider, as it also likes to
discard useful parts of error messages in its pretty display.
Where and how you get to the raw textual output varies between versions and
the classic/new GUI:
the button could be "Toggle Console View" next to "Build Output" or
"Toggle Tree View Mode" to the left of the errors.
2023.2/new GUI seems to default to showing both, which is a nice compromise.

For Rider,
[EzArgs](https://plugins.jetbrains.com/plugin/16411-ezargs) can replicate some
of the functionality of UnrealVS.

# Project structure

When you create a new Unreal project, you're given a set of options, notably
including a toggle between a BP or C++ project.
This has misled many people.
There's only one kind of .uproject: picking either is not a commitment but
simply a set of default settings and starter code that can be changed entirely.

BP-only projects do not contain a Source folder by default, but one can be added
by creating any C++ class from the editor.
Projects with a Source folder can return to being BP only by deleting Source and
editing the .uproject file (which is JSON) to no longer refer to the modules
that are now gone.

BP classes are considered content, but C++ classes are not.
As a result, the Unreal editor's "C++ Classes" folder in the Content Browser is
mostly useless and should be ignored.
This distinction is also relevant for loading them: BP classes are loaded like
any other content, while all C++ classes are immediately available when their
module loads, usually but not necessarily at project startup.

## C++ modules
{:.no_toc}

The generated Visual Studio solution and its accompanying vcxprojs are "fake",
and exist only for IDE compatibility.
They're not used in the build process aside from invoking Unreal's custom build
system, it's pointless to try and change compiler/linker/nmake settings in them.
Unreal's own build tool is the creatively-named UnrealBuildTool (UBT), which
makes heavy use of the UnrealHeaderTool (UHT) for code generation.

Your actual build files are the Target.cs and Build.cs files you get when you
create any starter C++ project template or add a C++ class to a BP-only
project.
[Here](https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-engine-modules)'s
the official documentation on them.

Target.cs files govern building your entire project.
Targets are things such as editor, standalone, or dedicated server.
These get mixed with the familiar "debug" and "release" configurations (but more
nuanced in Unreal) to form your overall list of project configurations.
See the documentation
[here](https://dev.epicgames.com/documentation/en-us/unreal-engine/build-configurations-reference-for-unreal-engine).

Build.cs files govern an individual module (.dll or static lib, C++20 modules
are not supported or used), not unlike .asmdefs in Unity.
An `Example` module would live in the Example folder under Source with an
`Example.Build.cs` file in it, its exported classes would be marked with
`EXAMPLE_API` (the dllimport/dllexport macro is automatically defined), and
other modules would reference it as `"Example"` in, e.g.,
their PublicDependencyModuleNames.
These all correspond to each other.

All of these .cs files in a project get compiled together, therefore you can,
e.g., define an intermediate base class or extension methods to apply common
build settings.
Approaches to manage this include adding extra functionality to Target.cs files
or defining an empty C++ module for nothing else but its C# functionality.

The Public/Private folder structure commonly seen is a convention directly
supported by the build system and the editor, but it's not strictly required.
Include paths default to a module's Public folder when it's added as a
dependency.
Otherwise it's a longer path relative to the Build.cs file.

## Plugins
{:.no_toc}

Modules may be part of your project directly or belong to plugins.
In addition to modules, plugins may also contain content (assets, blueprints)
just like your project, and have extra control for being toggled, and their
loading order in the corresponding .uplugin file.
Content-only plugins are also possible and all plugins may be installed directly
into the engine and shared between projects.

If a plugin exists in both the engine and a project, the one in the project
takes precedence.
This is useful to, e.g., apply fixes to engine plugins without having to
build an entire custom engine.
The drawback is that you're now responsible for keeping that plugin up-to-date
with future engine versions.

Generally, your project depends on generic plugins, but this is reversed in the
case of game feature plugins: these plugins are "backwards", and they depend on
your project to extend it without the project knowing.
They can, e.g., inject additional components into actors to run a seasonal event
(think Fortnite).

# [Compiling](https://xkcd.com/303/)

Launching your project via the .uproject file runs the `Development Editor`
configuration.
By default, Unreal will check if a build for this _exists_, but not if it's _up
to date_.
This is the source of many "why is my code not doing what it should?" issues,
serialization errors, possible data corruption, etc.
[This setting](/2022/09/27/tips-and-tricks.html#automatically-update-c-binaries)
helps to avoid running with outdated binaries.

To update C++ code while the editor is running, Unreal Engine comes with not one,
but two methods for live patching, and both are broken in different ways. 🎉

**It's recommended to have Live Coding ON and reinstancing OFF as safe settings,
which will prevent accidental Hot Reloads even if you do not use Live Coding.**
You can also explicitly disable Hot Reload.

Note that restarting the editor is not enough to get back to a clean state if
you used either of these.
You'll need to build (not rebuild, it's a waste of CPU time) **while Unreal is
closed**.

The only 100% reliable and stable method of building C++ is treating Unreal like
you would any other C++ program and compiling while it's **NOT** running.
Launching it for debugging from your IDE of choice is the best workflow for
this, as it avoids both outdated binaries (assuming your IDE is set to build
before running), and one method of triggering Hot Reload.

Don't use Unreal's built-in C++ class wizard, it tries to compile and load the
new class "live".
The editor needs to be closed in order to compile the new class safely anyway,
use an "add Unreal class" feature from your IDE instead.
[This section](#development-environment) has a few suggestions for both paid and
free addons with such functionality, in case you scrolled past it.
Ultimately, you're just creating two text files and compiling; anything will do.

Launching in a debugger has the added benefit of immediately catching rare
issues that sadly do happen during development.
I have regretted **NOT** running in a debugger on multiple occasions.
The Epic crash reporter will grab your call stack, but you'll lose runtime
state.
I'm personally using `DebugGame Editor` as my daily driver configuration, but
there's nothing wrong with `Development Editor` either.
It runs somewhat faster, helps keep your binaries up-to-date for .uproject
launches while you unlearn to do that, but it's less reliable for debugging.

## Hot Reload
{:.no_toc}

This is Unreal's own implementation that predates the Visual Studio hot reload
feature.

It's enabled by default on UE4 and suppressed by Live Coding on UE5.
Live Coding may be enabled on UE4 to suppress it there as well.

It can corrupt data in memory that may get persisted if you save any .uassets
after it happens.
This can corrupt these files permanently, requiring a revert or having to
recreate them from scratch to recover.
Copy-pasting nodes or data from one to another can carry the corruption with it.

Hot Reload can also cause random crashes and misleading results while debugging
due to the aforementioned corruption.

It kicks in automatically if you compile from your IDE while Unreal is running
unless disabled or suppressed.
If you accidentally did this, quit the editor and do not save anything when it
asks.
Technically, using Hot Reload while ensuring that no .uassets are ever saved
once it has happened in a running process is "safe" but also highly error-prone
and thus not recommended.

_Tools → Debug → Modules_ in the Unreal editor is an interface to Hot Reload
with the same issues.

## Live Coding (Live++)
{:.no_toc}

Enabled by default in UE5, disabled but available in UE4.
Windows only.
Also known as [Live++](https://liveplusplus.tech), named after the underlying
tech that Epic licenses.

With reinstancing on (default enabled in UE5, unavailable in UE4) it exhibits
corruption similar to Hot Reload, and the same recommendations apply.

With reinstancing turned off, it "merely" causes crashes and runtime corruption,
but your BPs/uassets remain safe.
This is often considered a worthwhile tradeoff for the speedhack-like iteration
benefits it provides.
At the first sign of things going wrong, quit the editor, build, and start it
anew.

## Unity
{:.no_toc}

UBT supports automatically-managed
[Unity builds](https://en.wikipedia.org/wiki/Unity_build), which is enabled by
default.
This dramatically speeds up clean rebuilds, and UBT's adaptive unity system
tries to strike a balance between this and not slowing down incremental builds.

However, unity can hide #include problems and even break otherwise-correct code
that, e.g., uses unnamed namespaces.
This can surface as code that's assumed to be correct "randomly" breaking due
to UBT rearranging what .cpp files belong to what unity translation unit.

Debugging these issues should start by turning unity off:
add `bUseUnity = false;` to your Build.cs, and your compiler/linker errors will
often make more sense and directly correspond to the real issue at hand.

There's a similar `bUseUnityBuild` setting available for Target.cs files that
affects **EVERYTHING**, including forcing a rebuild of the entire engine if
you're on a source build.
Only use this if you really mean it.

# Unreal type system

Unreal has its parallel, garbage-collected world that's heavily inspired by the
.NET type system.
`UCLASS` and `USTRUCT` correspond to the C# concept of a `class` and a `struct`
instead of C++'s.
UCLASSes are only passed around by pointer (at least in UFUNCTIONs) and are heap
only.
USTRUCTs are semantically passed by value even if they're references, Blueprints
will happily object slice them.

These types are required for anything the engine needs to reflect, such as
usage in Blueprints, automatic serialization, DYNAMIC delegates (see later), etc.

Their macros (also `UENUM`, `UFUNCTION`, `DECLARE_DYNAMIC_..._DELEGATE...`, etc.)
are #defined to nothing in C++ but parsed by UHT as part of the build process
and get an additional .h/.cpp pair generated per affected file.

It is automatically enforced to #include the `.generated.h` as the last item and
use `GENERATED_BODY()` within these types to inject extra members that the
engine needs.
If you ever see a similar three-word macro, such as `GENERATED_UCLASS_BODY`,
those are all obsolete and have been replaced by `GENERATED_BODY` in all cases.

This type system is limited in what it can do, e.g., it's enforced that
UFUNCTIONs can only pass UObjects by pointer or reference-to-pointer, and struct
pointers are disallowed.
Templates are essentially whitelisted to some built-in containers such as
TArray, TMap, etc.

UFUNCTIONs must have a unique parameter list, "overloads" need to have a
different name.

The Epic naming convention (USomething, ASomething, FSomething, ...) is enforced
for classes participating in this type system.

Sadly, these types are incompatible with namespaces.
The common workaround is to use namespaces for your "raw" types that can be in
one and a short (3-6 characters) prefix based on your project's or company's
name otherwise, to avoid name collisions with the engine and third-party plugins.

## UObject

UObjects are UE's .NET `System.Object` equivalent.
They exclusively live on the heap with a garbage collector taking care of them
and are usually passed around by pointers or references to pointers (think
`ref object`).
You mostly create them with `NewObject<T>` or `CreateDefaultSubobject<T>`.

The most important subclass of UObject is AActor, it even has its own class
prefix.
AActors have a
[complex lifecycle](https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-engine-actor-lifecycle)
that goes beyond mere garbage collection and reachability analysis, and even
come with their own creation function, `UWorld::SpawnActor`.

The garbage collector only "sees" members marked with `UPROPERTY`; anything not
reachable through them is considered unreachable and may be deleted.
USTRUCTs can be held by value as a UPROPERTY and in turn hold UObject\*
UPROPERTYs which would then be considered reachable, etc.
This is not reference counting; circular references are perfectly fine.

<sub>
Currently, objects can be explicitly marked as "pending kill", automatically
setting UPROPERTY pointers pointing to them to nullptr when they're garbage
collected.
This also applies to destroyed actors, but there's a newer GC operation mode
where this is no longer the case.
The new mode fixes some issues, such as formerly-different TMap keys now all
being nullptr in a container that does not support multiple keys.
This is opt-in with the inversely-named gc.PendingKillEnabled 0 setting, with
Lyra notably opting in ([/Script/Engine.GarbageCollectionSettings]
gc.PendingKillEnabled=False).
5.4 renamed the setting to gc.GarbageEliminationEnabled.
It's worth considering setting this if you're starting a new project, as it has
fewer surprises in store.
</sub>

See [Lyra](https://dev.epicgames.com/documentation/en-us/unreal-engine/lyra-sample-game-in-unreal-engine)
itself.

### Validity

There are non-nullptr UObject pointers that are considered invalid (such as, but
not limited to "pending kill" from the fine print above).

Only checking for nullness with, e.g., `if (SomeUObject)` will often work with
pending kill enabled, but not always.
This makes bugs caused by invalid pointers very difficult to reproduce and debug:
in this mode, nulling the pointers is also reliant on GC timing.

To check the validity of a raw UObject\*, use the global IsValid() or GetValid()
methods.
IsValid returns a bool, GetValid is a shortcut for
`IsValid(Input) ? Input : nullptr`.
This makes the following C++17 construct convenient to use if you want a
temporary copy of a pointer:
```c++
if (auto* Ptr = GetValid(Something->GetSomeUObject()))
{
    // Ptr may be used here
}
```

`if (auto* Ptr = SomeUObject; IsValid(Ptr))` is also completely fine, and has no
drawbacks if you prefer writing it that way.

### Object pointers
{:.no_toc}

There are smart pointers that let you reference UObjects from a "regular" C++
object that cannot have `UPROPERTY`s.
These all have `Object` in their names, such as `TStrongObjectPtr<T>`,
`TWeakObjectPtr<T>`, but **NOT** `TObjectPtr<T>`.
As an exception, TObjectPtr is not smart: it has no runtime effect in shipped
projects, where it is equivalent to a `T*` raw pointer.

TObjectPtr is used for UPROPERTYs only (writing `UPROPERTY()` is still required),
function parameters and return values are still `UObject*`.
This makes things complicated when, e.g., you need a `UObject*&`, but only have a
TObjectPtr.
The usual way engine code works around itself is by using a temporary UObject\*
for the reference, that is then assigned to the TObjectPtr.

Classes with only `Ptr` such as `TSharedPtr` or `TUniquePtr` are reinventions of
their STL counterparts (with varying levels of quality...) and work with common
C++ objects.
`Ref`s are the same but disallow being nullptr.

Object and non-object pointers may not be mixed in either direction.
`TSharedPtr<UObject>` and `TWeakObjectPtr<FVector>` are both invalid.

### Casts

Unreal is compiled without exceptions or RTTI, so dynamic_cast is not available.
It's #defined in engine code as a horrendous macro hack that's better avoided
entirely.
`Cast<T>` is its replacement for UObjects, with `CastChecked<T>` behaving like
static_cast with additional debug checks that are free in shipping.
With these, you only write `T` instead of `T*` as the template parameter.

### Class Default Objects (CDO)

Unreal will default construct (for the sake of discussion, this includes
constructors taking `const FObjectInitializer&`) every reflected type to query
its defaults and keep it in memory, accessible via `UClass::GetDefaultObject`.
This will explain most of the "unexpected" constructor calls you might see.

For this reason, USTRUCTs and UCLASSes must come with a default constructor.
You can use other constructors with USTRUCTs, but this is not the case with
UCLASSes where you have virtually no control over what constructor gets called.

The replacement for copy constructors is the `Template` parameter to NewObject,
and there's no direct replacement for parameterized constructors.
Common workarounds include an Init method, a static Create method, or
SpawnActorDeferred for actors.

Subclasses, UPROPERTYs, etc., are serialized as deltas from CDOs.
This is how the "revert arrow" works in the editor's Details panel.
This serialization process is brittle and requires you to get some things right,
detailed below.

Blueprint classes also have CDOs, but these are only created when the BP is
loaded.

#### Default subobjects, components
{:.no_toc}

Classes might have default subobjects, that is, additional objects that get
created together with them when instantiated.
The most common practical usage of this is an actor constructor creating its
components.

For these, use `CreateDefaultSubobject<UComponentClass>("PropertyName")` instead
of NewObject, and store the result in a **Visible** UPROPERTY.
Do not use spaces in these FName parameters, doing so triggers bugs.

Never use **Edit** specifiers (EditAnywhere, etc.) for these properties.
That leads to incorrect serialization and BP corruption.
The component's properties will still be editable when you use Visible; it's the
pointer itself that's affected here.
For the same reason, BlueprintReadWrite is rarely useful for component
properties, and BlueprintReadOnly should be preferred (but at least getting this
wrong won't directly corrupt your files).

You'll often see the TEXT() macro used for CreateDefaultSubobject in
questionable learning resources and tutorials.
This is a pure waste of CPU if your FName is ASCII (as it should be) and
should not be used.
Not even Epic employees are fully aware of how this works.
TEXT() is for FString literals.

Constructors use `->SetupAttachment()` to build component hierarchies.

At runtime, components are made by
NewObject+RegisterComponent+AddInstanceComponent.
SetupAttachment may be used only before calling RegisterComponent; after that,
components are attached with the "usual" functions that are also available in BP.
Engine code randomly forgets to call AddInstanceComponent, which leads to, e.g.,
your component existing but not being visible in the inspector.

A BP Construction Script's C++ equivalent is the OnConstruction method, not the
constructor.
Components are made with NewObject in this case, and to match BP behavior, you'll
need to set this flag manually:<br>
`YourNewComponent->CreationMethod = EComponentCreationMethod::UserConstructionScript;`

There's another "construction script" concept in the engine, called a simple
construction script.
This is what contains BP-added components in actor blueprints.
A BP actor's CDO does **NOT** contain components added this way, unlike C\+\+
CreateDefaultSubobject.

## UClass

UClass (not to be confused with `UCLASS`) is the closest equivalent of .NET's
`System.Type`.
UClass itself is used the most often, but there are multiple related classes
that represent types, such as UStruct, UBlueprintGeneratedClass, etc.

The two most common ways of obtaining these objects are `T::StaticClass()` and
`Obj->GetClass()` (cf. typeof and GetType() in .NET).
Many engine functions that take a UClass* parameter come with template
overloads that let you retain some amount of type safety, e.g., `Foo<UExample>()`
replaces `Cast<UExample>(Foo(UExample::StaticClass()))`.

### TSubclassOf
{:.no_toc}

This wrapper is often used instead of UClass* to limit the potential values
that may be set or used.
It implicitly converts from/to UClass*.

In the editor, UPROPERTYs of this type will limit what options are offered.
At runtime, if a TSubclassOf contains a "wrong" class, it will be returned as
nullptr:

```c++
TSubclassOf<UObject> Example1; // Defaults to no class
TSubclassOf<UObject> Example2 = UObject::StaticClass(); // Straightforward
TSubclassOf<AActor>  Example3 = UObject::StaticClass(); // Still OK to do
check(Example1 == nullptr);
check(Example2 != nullptr);
check(Example3 == nullptr); // UObject is not a child of AActor
```

## Delegates

.NET's delegates and events (member function pointers + their `this`) also come
in Unreal flavor.
[This](https://benui.ca/unreal/delegates-intro/) is a good overview.

Dynamic delegates work based on the reflection system and can only call
UFUNCTIONs.
This limitation is required for them to work in BP.
These are declared with the various `DECLARE_DYNAMIC_..._DELEGATE_...` macros
and can be stored in UPROPERTYs, passed into UFUNCTIONs, etc.

Dynamic multicast delegates can be UPROPERTY(BlueprintAssignable) and unicast
ones can be taken and returned from UFUNCTIONs, among other things.

Regular delegates come with additional features over something like
std::function+std::bind, such as being able to weakly reference a UObject
and auto-unbind if it gets garbage collected.
There are `DECLARE_DELEGATE_...` macros (without DYNAMIC), but these are
practically obsolete as all of them resolve to something like
`using FYourDelegate = TDelegate<void(int)>;` (or TMulticastDelegate, etc.) that
you can write directly in a cleaner syntax or not define an alias at all.
The macros remain required for dynamic delegates as they involve generated code.

These macros have issues with types having commas in them, such as TMaps.
The fix is trivial in the non-dynamic case (don't use the macros), but dynamics
require wrapper USTRUCTs to avoid the comma.

Events are worth mentioning (`DECLARE_EVENT`).
They used to provide extra encapsulation compared to a delegate field but are
obsolete and have been replaced by delegates according to a comment in the
engine's source code.
The implementation of these macros has changed from how it used to work and the
additional encapsulation is no longer provided.

### Delegates and CDOs
{:.no_toc}

Do not bind to delegates in UCLASS/USTRUCT constructors.
That will include the CDO that probably shouldn't respond to broadcasts.

UPROPERTY delegates get serialized, which is an even bigger problem as it leads
to corruption and weird behavior.
Use an appropriate overload, such as BeginPlay or PostInitProperties, to bind
early at runtime instead.

# Blueprint interop

BP can access the reflection information built by UHT and "sees" types and
members marked with the various UMACROs.
There's often an extra specifier required, such as UPROPERTY(BlueprintReadWrite)
or UFUNCTION(BlueprintCallable), to expose something to BP.

Conversely, C++ is compiled before BP is even loaded and cannot access anything
without brittle, slow, and string-based runtime reflection hacks of desperation.
You might have guessed that this approach is best avoided.

The main way for the two languages to talk to each other is for C++ to offer
values or hooks that BP uses or implements.

There are multiple competing naming conventions indicating that something is
intended for BP.
The most commonly used is `K2_Something()`, K2 standing for Kismet 2.0.
Some other, less common ones include `BP_Something()`, `ReceiveSomething`, and 
`NotifySomething`.
These are often paired with a DisplayName specifier in their UFUNCTIONs to get
sanitized for BP.

<sub>
Kismet was the original visual scripting language of Unreal Engine 3.
It only worked for levels, and its direct UE4 equivalent is a level blueprint
(ALevelScriptActor).
It still exists in UE5 but has issues when World Partition is involved.
</sub>

## Properties

UPROPERTY is the main method of sending runtime data to and from blueprints.
A helpful trick is having UPROPERTYs as `private` but adding the
`AllowPrivateAccess` meta-specifier, which will expose them anyway.
This will prevent direct access to the field from C++, enforcing the use of your
accessors, and can be treated as something like "protected for BP only".

Accessor UFUNCTIONs can also be provided (along with full encapsulation),
but use BlueprintGetter and BlueprintSetter for this to keep your designers
happy and not leak "C++-isms" into BP.

## Types

BP has its own take on classes, enums, structs, etc.

Classes are great, use them as much as you want, but BP enums and BP structs
have bugs and are considered essentially unportable to C++.
Define a C++ UENUM or USTRUCT instead from the get-go.

### Asset references
{:.no_toc}

One of the most important usages of BP is to provide assets and configuration
values to UPROPERTYs.
This is most commonly done through literal blueprints (e.g., actor subclasses),
but there are other similar systems to achieve this, such as UDeveloperSettings.

Raw pointers and TObjectPtr are force loaded before you have a chance to read
them, while soft pointers (TSoftObjectPtr) are loaded on demand by your code.
Soft pointers store a path to the asset and act as weak pointers after they've
been loaded.

They're great for reducing editor startup time and ingame loading screen times:
you can issue a load well in advance and only do a last-resort synchronous block
when your player was unexpectedly quick.
For instance, you could have an invisible trigger leading up to a boss room,
that started loading the textures of the boss while the player is traversing the
last corridor.

Note that UPROPERTYs are set **AFTER** your C++ constructor has returned.
Therefore, you cannot read their values just yet.
Various callbacks are invoked after properties have been written, such as
OnConstruction, PostInitProperties or PostLoad.
You can use these to react to the new values.
The editor has additional callbacks for "live" editing of properties in the
Details panel once an object has been constructed and loaded, such as
PostEditChangeProperty or PostEditChangeChainProperty.

#### UBlueprint
{:.no_toc}

It can be a little confusing what UBlueprint even is, since blueprints can have
a parent class that you decide, and that class doesn't inherit from UBlueprint
at all.

In the case of `BP_ExampleActor`, that would be the UBlueprint asset that
contains the class `BP_ExampleActor_C` actually inheriting from AActor.

Data assets (UDataAsset, UDataTable, etc.) are saved directly.

## Functions

UFUNCTION(BlueprintImplementableEvent) functions are only declared in .h files
and not implemented in C++ but may be called from there.
They will become an overrideable function (or "event" if not returning anything,
i.e., a <!--♪-->coroutine<!--♫-->) in BP.

UFUNCTION(BlueprintNativeEvent) functions may have a C++ implementation, which
is nearly always virtual.
For `UFUNCTION(BlueprintNativeEvent) void Example();`, implement
`void ClassName::Example_Implementation(){}`.
This naming convention is enforced.

Always call `Example()` instead of its _Implementation, or you'll bypass BP.
An exception to this is calling `Super::Example_Implementation()` in overrides.

Using a BIE instead of a BNE can be beneficial if you want to ensure that BP
cannot possibly forget to call the parent implementation.
Have a BlueprintCallable function (Example) do its mandatory C++ thing, then
call a BlueprintImplementableEvent version of itself (K2_Example) to provide
the "override" opportunity for BP.
This is how AActor::BeginPlay() itself is written, for instance.

UPROPERTY(BlueprintAssignable) delegates are also implementable in BP, despite
being properties.
The BP event will be bound to the delegate automatically.

Going in the opposite direction, UFUNCTION(BlueprintCallable) and
UFUNCTION(BlueprintPure) let C++ functions be called from BP.
Callable functions have an "exec" (white) pin in BP and run only if that
line transfers control, while Pure functions don't have one.
This invokes the idea of purity from languages such as Haskell, but in practice,
it **only** means that pure functions are called every time data is pulled out
of them.
BlueprintPure is frequently misused for functions that are technically impure.

`const` functions default to being pure, but can be marked impure with
UFUNCTION(BlueprintPure=false).
Otherwise purity is opt-in.

The C++->BP and BP->C++ specifiers may be combined, e.g.,
`UFUNCTION(BlueprintImplementableEvent, BlueprintCallable)` is valid.

## Interfaces

Unreal has a bizarre implementation of interfaces.
In concept, they're similar to C# interfaces (UObjects may only have one
UObject parent class but may inherit any number of IInterfaces, `#if CPP` hacks
aside), but the way you use or call them is inconsistent depending on whether
they're meant to be used in BP or not.

Purely C++ interfaces (`NotBlueprintable` or
`Meta=(CannotImplementInterfaceInBlueprint)`) work with `Cast<T>` and regular
`->Function()` calls, as you would expect.

If BP is involved though, it becomes entirely different: `->Function()`s will
assert, Cast will not work, and you'll need generated static helpers.
Note the usage of both the U and I halves of the interface:
```c++
// Instead of Thing->Example(1, 2, 3)
if (Thing->Implements<UMyInterface>())
    IMyInterface::Execute_Example(Thing, 1, 2, 3);
```

This will correctly call C++ or BP regardless of where the interface was
implemented.
Use TScriptInterface to hold objects that might implement an interface this way
since they are not IMyInterface\* in this case.

# Threading

Epic decided to be friendly to designers by forcing all game code to one thread,
called the game thread.
This is the thread that the garbage collector pauses when collecting, the one
that runs most engine Tick()s, etc. and is key to many threading scenarios.

There are other important threads named by the ENamedThreads "enum", which is a
namespace, a common workaround for UENUMs before `enum class` was added to C++.

The most common way to interact with this system is with the Task Graph, which
can be used via the AsyncTask() wrapper, or TGraphTask for more explicit control.
This is great for small tasks (usually within a frame) but avoid overloading
these threads, or you'll compete with the engine.

There's a newer frontend to the low-level tasks system called
[UE\:\:Tasks](https://dev.epicgames.com/documentation/en-us/unreal-engine/tasks-systems-in-unreal-engine),
which is relatively unused by engine systems, mostly due to its relative youth.
It has a better API for chaining, dependencies, etc.

Full threads are usually made with the FRunnable interface and launched with,
e.g., `FRunnableThread::Create`.
This is usually recommended for long-running tasks (a second or more).

## Game thread and UObjects
{:.no_toc}

Barring a few exceptions, UObjects should be considered to live on and be owned
by the game thread.
The garbage collector expects to be able to scan their UPROPERTYs and delete
them without any notice or synchronization.
This will only ever happen cooperatively on the game thread, meaning that if
your code is running, you own the entire thread and need not worry about local
"un-propertied" raw pointers to UObjects.
Returning or suspending (return, co_await, co_yield, co_return) a function gives
it a chance to run, though.

This setup naturally leads to threading setups where expensive operations are
offloaded to other threads, but the results are applied on the game thread in,
e.g., an AsyncTask.

It is your responsibility to ensure this will run correctly.
Common techniques include arranging that the UObject will be referenced and not
GC'd while the off-thread computation is running, or checking if it's gone
before attempting to pass results back (giving a TStrongObjectPtr or a Weak one
to the other thread can achieve this), canceling the task and blocking until its
completion in the UCLASS's destructor, etc.

# Miscellaneous extras

Useful things that don't really fit any other category.

## Logging

You'll see the legacy UE_LOG macro being used in lots of online resources.
It's using legacy C varargs with all the issues that it comes with: if you're
seeing prefix \* operators being used with FStrings, that's part of the
workarounds it requires to be even able to pass the parameter at all.

It has been replaced by the C\+\+-based, type-safe UE_LOGFMT that uses a
.NET-inspired syntax:

```c++
UE_LOGFMT(LogTemp, Display, "My values: {0} {1}", 123, MyString);
```

Although UE_LOGFMT is strictly superior, it's still weirdly lacking support for
some of the engine's own types (e.g., FVector).

I have [written](/2022/04/28/better-ue_log.html)
[some](/2022/05/03/better-ue_log-2.html)
[articles](/2023/03/21/better-ue_log-3.html) on this topic, and published a
[sample library](https://github.com/landelare/llog) that's capable of
conveniently and type-safely logging most Unreal types, even in STL containers.
It's published under a very permissive free software license in case you only
want to grab the formatter part of it.

## Assertions

Unreal comes with a rich set of assertion macros, none having `assert` in their
names.
[Here](https://dev.epicgames.com/documentation/en-us/unreal-engine/asserts-in-unreal-engine)
is the official documentation.

It often surprises people that the "Slow" assertion macros do **NOT** run in
DebugGame configurations, only full Debug.

You can "fix" this if desired by adding this to your Target.cs:
```c#
if (Configuration <= UnrealTargetConfiguration.DebugGame)
    ProjectDefinitions.Add("DO_CHECK_SLOW=1");
```
Or per module into a Build.cs:
```c#
if (Target.Configuration <= UnrealTargetConfiguration.DebugGame)
    PublicDefinitions.Add("DO_GUARD_SLOW=1");
```

## Subsystems

Unreal's take on singletons.
These inherit from `USubsystem`, and you, in turn, inherit from one of the
specialized subclasses, such as `UWorldSubsystem` or `UGameInstanceSubsystem`.
These get created and destroyed on demand together with the thing they're a
subsystem of, with convenient getters (`GetSubsystem<T>` that you can wrap in
a static getter if you wish).

In BP, these receive pure getter nodes, but you might want to provide a BP
function library with statics that get the subsystem individually to reduce the
spaghetti.

BP subsystems require hacks to even function and are considered infeasible.
Assets may be referenced through a UDeveloperSettings subclass.

[This plugin](https://github.com/aquanox/SubsystemBrowserPlugin) extends editor
functionality for them.

## Gameplay Ability System (GAS)

Unreal's built-in support for "game" things such as skills, stats, status
effects, etc., in a generic system.
Used by Epic in games such as the Lyra starter game and Fortnite itself.

It's officially not released as a feature but still shipped with the engine on a
"here, it's useful, but you get no support" basis.
It's great for single- and multiplayer both, but especially powerful in
multiplayer, simply because it has more things to deal with on your behalf.

[This](https://github.com/tranek/GASDocumentation) is one of the best pieces of
community documentation for it.

### Gameplay Tags
{:.no_toc}

FNames on steroids that are hierarchical and can be defined in advance so that
you don't have to keep typing strings and hoping you don't get one of them wrong.
Although it's commonly used in conjunction with GAS, it's an independent system.

[Here](https://dev.epicgames.com/documentation/en-us/unreal-engine/using-gameplay-tags-in-unreal-engine)'s
the official documentation.

## BindWidget
{:.no_toc}

This is a UPROPERTY specifier that lets you create a C++ UMG widget with its
sub-widgets being visually created in the designer and "arriving" in the
UPROPERTY by name.
[Here](https://benui.ca/unreal/ui-bindwidget/)'s an entire article on it.

# Useful links for reference

|[Debugging features](https://dev.epicgames.com/community/learning/tutorials/dXl5/advanced-debugging-in-unreal-engine)|[String-like classes](https://dev.epicgames.com/documentation/en-us/unreal-engine/string-handling-in-unreal-engine)|[Assorted tips and tricks](/2022/09/27/tips-and-tricks.html)|
| :-- | :-- | :-- |
|[All UPROPERTY specifiers](https://benui.ca/unreal/uproperty/)|[All UFUNCTION specifiers](https://benui.ca/unreal/ufunction/)|[All UCLASS specifiers](https://benui.ca/unreal/uclass/)|
|[All USTRUCT specifiers](https://benui.ca/unreal/ustruct/)|[All UPARAM specifiers](https://benui.ca/unreal/uparam/)|[All UENUM and UMETA specifiers](https://benui.ca/unreal/uenum-umeta/)|
