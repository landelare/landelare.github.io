---
title:  "Unity to Unreal starter pack"
excerpt: "A small collection of things that I wish I was told when I jumped ship."
last_modified_at: 2022-12-24
---

It seems that
[recent](https://www.pcgamer.com/unity-is-merging-with-a-company-who-made-a-malware-installer/)
[news](https://kotaku.com/unity-john-riccitiello-monetization-mobile-ironsource-1849179898)
has caused another wave of people leaving Unity for Unreal or at least
considering a switch and checking things out "on the other side" to see if it's
worth losing access to all those assets bought over the years.
If you're reading this, chances are you're one of these people. :)

As someone who had to make a similar call a few years ago, I can say that even
accounting for the months "lost" on porting and having to learn new tools, it
was still worth it. (Obviously the project was not near shipping, YMMV.)

# UE4 or UE5?

As of writing, 5.0.3 is the latest stable version.
Speaking of versions, there is no TECH/LTS split in Unreal.
Versions with "preview" in their names are just that, betas, anything else
numbered is stable (generally, more stable than a Unity LTS version).

If you're reading this from the future with 5.2 or beyond, the answer will
probably be an obvious "5", but in 5.0 there are a few instances where 4 should
still be considered, namely AR/VR and mobile.
5.0 was focused on getting the shiny high-end stuff out, and these have suffered
a little.
Several features such as One File Per Actor that are marketed for UE5 are in
fact UE4 features, and UE4 for the time being remains a very solid platform to
ship games if you're comfortable with committing yourself to never updating.

If you do plan to update eventually then don't bother and start with 5.

# Development

Unity has a very programmer-centric development style compared to Unreal, with
Bolt being a relatively recent addition to the engine.
Even designers often deal with C# code and everything revolves around
`MonoBehaviour`s one way or another.

Unreal is not just more artist-focused, but the entire architecture is different
in a way that I don't think the
[official documentation](https://docs.unrealengine.com/4.27/en-US/Basics/UnrealEngineForUnityDevs/)
explains well enough.
Definitely read that page, but then please come back here and let me tell you
about the things that took me some time to realize. :)

## Game code

In Unity, you write scripts and attach them to `GameObject`s that don't really
do anything by themselves, all functionality comes from components.
There is also a very clean line drawn between "your code" and "engine code".

In Unreal, your C++ code is on equal footing with the engine, you're extending
the architecture itself with your game-specific classes.
Start **un**learning to make everything a component and organize your code in
classes that make sense in your game world.
Don't try and cross-reference components across different actors, that's rare to
do in Unreal.
You can directly have logic in your actors unlike in Unity, use them directly.

Because your C++ is going directly into the engine, the way you compile code is
also different.
Normally you leave Unity running, change some C#, reload the script runtime, and
test your changes.
Unreal should be treated more like as if you were writing a command-line program:
you're working on the engine itself instead of being limited to a sandbox.
The sandbox is BP and it provides a similar rapid iteration workflow.

There are QoL tools that attempt to do some form of hot reloading, one of which
is [broken](http://www.hotreloadsucks.com), and the
[other one](https://docs.unrealengine.com/5.0/en-US/using-live-coding-to-recompile-unreal-engine-applications-at-runtime/) is unstable.
Feel free to use Live Coding, but you'll need some time to get used to working
with this style of development—you'll get some number of successful reloads
(anywhere from 0 to many) before you need to ultimately close the Unreal Editor,
compile your code "cleanly", and start again.
This does not mean a full rebuild, but the only 100% reliable compilation method
is building your C++ code with the editor **closed**, **then** starting it anew.

### Getting started

If you don't already know C++, these are good resources for learning it:<br>
[https://stackoverflow.com/questions/388242/the-definitive-c-book-guide-and-list](https://stackoverflow.com/questions/388242/the-definitive-c-book-guide-and-list)<br>
[https://www.learncpp.com](https://www.learncpp.com)<br>

This (underwhelmingly-titled) course from Epic will introduce you to the various
`USOMETHING` macros that you're expected to use:<br>
[https://dev.epicgames.com/community/learning/courses/KJ/converting-blueprint-to-c/](https://dev.epicgames.com/community/learning/courses/KJ/converting-blueprint-to-c/)

In your first month or two, I'd also recommend keeping
[https://tackytortoise.github.io/2022/06/24/common-slacker-issues.html](https://tackytortoise.github.io/2022/06/24/common-slacker-issues.html)
on speed dial.

Using ReSharper or Rider is also highly recommended, getting autocomplete for
the magic macros is very nice as a beginner.
Visual Assist used to be another solid choice for Unreal development.

## C++ "vs" Blueprints

Most people with a Unity mindset think of this as a choice that has to be made,
and default to C++ as the _obvious_ choice.
That's a mistake, the correct choice is **both**.

BP is not just the node graph programming language, it's a collection of systems
built on the reflection system that also deals with saving values on your actors
("prefabs"), upgrading components in derived classes, serializing data
("ScriptableObjects"), etc.

I found it useful to default to making actor classes in pairs—immediately create
a BP subclass for your C++ class and only ever use the BP one.
Empty classes are virtually free and you'll have them ready to go when you need
to add something in either language.
The situation becomes a little more complicated with subsystems.
If you find yourself thinking of using constructor helpers (don't!) and there's
no obvious BP escape route, look into `UDeveloperSettings`.

Other than this, BP is great for anything that involves logic running across
multiple frames (their name for async stuff is "latent", nodes with a 🕒 icon).
I'm ~~shamelessly promoting~~ developing a plugin that adds
<!--♪-->coroutines<!--♫--> to UE5 (see the top of this page), but BP remains the
only officially-supported way that doesn't involve callback hell.
Timeline nodes are very convenient for interpolation and quick one-off
animations.
You can easily run them by calling a custom `BlueprintImplementableEvent` from
C++.

I won't attempt to suggest a "correct" balance of BP and C++ because it depends
on team skills and preferences.
Being C++ heavy, BP heavy, or a mix closer to half-and-half all have their
individual benefits and drawbacks. 100% of either is a Very Bad Idea™.

# Overall engine "feel"

Let's face it, C# is amazing and a serious benefit of Unity compared to Unreal.
However, you have to use Unity for that, which is becoming more and more of an
issue.
This is the part where Unreal shines and makes it worth tolerating C++ and BP.

A **lot** of things that you would buy Unity assets for or develop yourself are
just... there already. Very often you need but to click a checkbox.
Especially when you're new, it's worth spending ≈10 minutes trying to find an
engine implementation of whatever you're trying to do, because it will usually
be there.
(And, you have full source code access as standard!)

There is no messing around with packages and their incompatibilities.
There is no SRP fragmentation hell (there is an alternative forward-rendering
pipeline which is compatible with the deferred one if you don't access the
GBuffer).
A character controller (both manual and AI) comes out of the box. It's a
built-in class, which is great because virtually every marketplace asset will
support it.
There is multiplayer support.
There are obviously the shiny new features such as Nanite and Lumen.
There is an entire generic gameplay ability system that comes with skill levels,
cooldowns, status effects, etc.
The list goes on.

Epic Games actually makes and ships games, and you can't help but notice all the
small QoL things that permeate the engine and are completely missing from Unity.

## Drawbacks

Unreal is not perfect and there are some things that are worse and require
additional discipline or forethought compared to them Just Working™ in Unity.

A lot of what would be YAML files in Unity are binary files that are a nightmare
to version control. Things are mainly referenced by paths instead of GUIDs that
lead to hacks like "redirector" assets or configuration entries to keep
references alive when you move or rename something.
You'll need to avoid merges with a locking workflow using something like Git LFS
or P4.
One File Per Actor provides some relief for .umaps, use it (4.26+)!

Unity uses quaternions throughout with helper functions such as `Euler` to
convert from designer-friendly representations, making it relatively
straightforward to avoid gimbal lock.
Unreal has `FRotator` as an entirely separate type, and you'll need to go out of
your way to make sure everything uses the `FQuat` overloads.
BP doesn't even have quaternions exposed by default.

Epic ships a lot of Not-Invented-Here replacements for standard stuff that are
incompatible, buggy, or both, for example `TUniquePtr` instead of
`std::unique_ptr`, `TFunction` instead of `std::function`, `TVariant` instead of
`std::variant`, etc.
There is no universal best pick for these (see [here](/2022/06/23/epic-conventions.html)),
some of them are genuinely better and good to have, and Epic themselves have
started abandoning others like `TAtomic`.

Speaking of bugs, although this is unfortunately equivalent to Unity, there are
some features that are more or less broken.
`UChildActorComponent`s sound like a nice replacement for prefabs, but don't use
them (at all).

Strings come in three flavors with their separate macros for literals.
`FName` needs `""` (most often), `FString` needs `TEXT("")`, `FText` needs
`LOCTEXT`/`NSLOCTEXT`/`INVTEXT`, but some other combinations also work (and
waste CPU).
Epic code is not consistent when it comes to using these correctly.

Constructors make `UObject`s differently compared to when they're made at
runtime.
This is a small minefield that you have to correctly navigate every time:
components need `Visible` `UPROPERTY` specifiers, not `Edit` (even for editing),
and when creating them at runtime you need to call both `RegisterComponent` and
`AddInstanceComponent`.
Why these are not one function call is a mystery, and even engine code randomly
forgets calling the latter.

The convention for engine units is 1u=1cm, and the coordinate system uses +X for
forward.
This means most digital content creation tools need special settings to export
assets correctly (and often you'll see hacks like characters being rotated 90°
in a blueprint because the meshes were imported from Maya).

Some parts of BP are best to be avoided completely, structs and enums are
infamous examples.

## Tips and tricks

**This section has been moved to a
[separate article](/2022/09/27/tips-and-tricks.html).**
