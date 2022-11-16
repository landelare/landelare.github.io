---
title: "Assorted tips and tricks"
excerpt: "A small collection of hidden settings."
last_modified_at: 2022-11-16
---

This will be nothing new for you if you've read
[Unity starter pack](/2022/07/16/unity-starter-pack.html), but these turned out
to be more popular among people who are otherwise not transitioning from Unity.

# Automatically update C++ binaries
If your team is small enough that you can have everyone install Visual Studio,
add this to DefaultEditorPerProjectUserSettings.ini in Config (create it if it
doesn't exist yet):
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

# Restore camera after Play-In-Editor

By default the camera stays where you were when you ended PIE.
If you prefer, you can change it to return where it was by adding this to
DefaultEditorPerProjectUserSettings.ini:
```
[/Script/UnrealEd.LevelEditorViewportSettings]
bEnableViewportCameraToUpdateFromPIV=False
```

PIV was the old name of PIE: Play-In-Viewport.

# Opt out of data collection

These go into DefaultEditor.ini:
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

# Greatly reduce shader compilation

_UE5.1 update: This is now turned on by default._

DefaultEngine.ini:
```
[SystemSettings]
r.ShaderCompiler.JobCacheDDC=1
```

This makes shaders get compiled on demand instead of thousands upfront.
