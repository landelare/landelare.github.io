---
title: "Assorted tips and tricks"
excerpt: "A small collection of hidden settings and other features."
last_modified_at: 2024-08-29
---

This collection of tricks used to be part of the
[Unity starter pack](/2022/07/16/unity-starter-pack.html), but these turned out
to be more popular among people who are otherwise not transitioning from Unity.

# Screen-space rotation widget

Users transitioning from Unity often miss this gizmo,
and for some reason it's off by default.

In Editor Preferences, look for `Enable Screen Rotate` (there's a search bar),
or add this to DefaultEditorPerProjectUserSettings.ini in Config (create it if
it doesn't exist yet):

```
[/Script/UnrealEd.LevelEditorViewportSettings]
bAllowScreenRotate=True
```

# Automatically update C++ binaries
If your team is small enough that you can have everyone install Visual Studio,
add this to DefaultEditorPerProjectUserSettings.ini:

```
[/Script/UnrealEd.EditorLoadingSavingSettings]
bForceCompilationAtStartup=True
```

This makes it so that opening the project from the .uproject file compiles C++,
avoiding having to make and distribute builds across the team.
Designers and artists need only to install the required build tools, forget they
ever existed, and enjoy using the correct binaries with zero additional effort.
Otherwise, note that .uproject uses the `Development Editor` configuration,
which is optimized. If you've been working with `DebugGame Editor` it might be
outdated. For developers the easy "fix" is to always launch the editor from VS.

If you have settings in Saved, those take precedence so check there if it
doesn't seem to have an effect. That's EditorPerProjectUserSettings.ini without
Default in its name.

<sup>
There's a bug between 5.0 and 5.3: the popup that would show compilation
progress does not appear.
Your project will look like it's not launching, but you can observe high CPU
usage while your code is otherwise compiling in the background normally.
You will get either the editor splash screen, or a message box with an error
eventually.
This bug did not exist in UE4 and was fixed in 5.4.
</sup>

# Restore camera after Play-In-Editor

By default the camera stays where you were when you ended PIE.
If you prefer, you can change it to return where it was by adding this to
DefaultEditorPerProjectUserSettings.ini:
```
[/Script/UnrealEd.LevelEditorViewportSettings]
bEnableViewportCameraToUpdateFromPIV=False
```

PIV was the old name of PIE: Play-In-Viewport.

# Debug memory-related crashes

Launch with `-stompmalloc` (use
[UnrealVS](https://dev.epicgames.com/documentation/en-us/unreal-engine/using-the-unrealvs-extension-for-unreal-engine-cplusplus-projects)
or [EzArgs](https://plugins.jetbrains.com/plugin/16411-ezargs)) and issue the
`gc.CollectGarbageEveryFrame 1` console command.

Attempt to reproduce your problem, you should have a much cleaner call stack
leading to the crash, often directly telling you what went wrong.

# Find and print references to UObjects

If you have a UObject that should have been garbage collected, but it isn't,
calling `FReferenceChainSearch::FindAndPrintStaleReferencesToObjects` on it will
log what other objects are currently referencing it, preventing its deletion.

Unreal itself will call this if you end a Play-In-Editor session and strongly
reference something (e.g., with a TStrongObjectPtr) that the engine wants gone.

# Opt out of data collection

These are also editor options but since you probably have
DefaultEditorPerProjectUserSettings.ini open already:
```
[/Script/UnrealEd.AnalyticsPrivacySettings]
bSendUsageData=False

[/Script/UnrealEd.CrashReportsPrivacySettings]
bSendUnattendedBugReports=False
```

If you want to further reduce the amount of network traffic that the editor
generates, add this to DefaultEngine.ini but note that this can impact
functionality:
```
[/Script/UdpMessaging.UdpMessagingSettings]
EnabledByDefault=False
EnableTransport=False

[/Script/TcpMessaging.TcpMessagingSettings]
EnableTransport=False
```

# Use your entire CPU to compile shaders

By default, Unreal throttles itself to around 80% of your CPU to compile shaders.
To make it use all your CPU to compile faster, add this to DefaultEngine.ini:
```
[DevOptions.Shaders]
NumUnusedShaderCompilingThreads=0
PercentageUnusedShaderCompilingThreads=0
WorkerProcessPriority=1
```
