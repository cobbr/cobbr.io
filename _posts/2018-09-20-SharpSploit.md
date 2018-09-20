---
layout: post
title: "Introducing SharpSploit: A C# Post-Exploitation Library"
date: 2018-09-20 07:00:00 -0600
tags: SharpSploit .NET dotnet C#
---

Today, I'm releasing [SharpSploit](https://github.com/cobbr/SharpSploit), the first in a series of offensive C# tools I have been writing over the past several months. [SharpSploit](https://github.com/cobbr/SharpSploit) is a .NET post-exploitation library written in C# that aims to highlight the attack surface of .NET and make the use of offensive .NET easier for red teamers.

[SharpSploit](https://github.com/cobbr/SharpSploit) is named, in part, as a homage to the [PowerSploit](https://github.com/PowerShellMafia/PowerSploit) project, a personal favorite of mine! While [SharpSploit](https://github.com/cobbr/SharpSploit) does port over some functionality from [PowerSploit](https://github.com/PowerShellMafia/PowerSploit), my intention is **not** at all to create a direct port of [PowerSploit](https://github.com/PowerShellMafia/PowerSploit). [SharpSploit](https://github.com/cobbr/SharpSploit) will be it's own project, albeit with similar goals to [PowerSploit](https://github.com/PowerShellMafia/PowerSploit).

### The Appeal of C\#

There seems to be a trend developing on the offensive side of the security community in porting existing PowerShell toolsets to C#, particularly with the recent releases from my SpecterOps teammates, including: [@harmj0y](https://twitter.com/harmj0y)'s [GhostPack](https://github.com/GhostPack) toolset and [@0xthirteen](https://twitter.com/0xthirteen)'s [SharpView](https://github.com/tevora-threat/SharpView). And [SharpSploit](https://github.com/cobbr/SharpSploit) is another piece to that puzzle. With the added security features in PowerShell (ie. ScriptBlock Logging, AMSI, etc.), it makes sense that red teamers are investing in other options. And C# is the logical next step from PowerShell, seeing that they both are based on the .NET framework and porting toolsets from PowerShell to C# is fairly easy to do.

However, C# does not come without it's own set of issues from an offensive perspective. It certainly seems as if optics into .NET [are on the way](https://twitter.com/mattifestation/status/1034197284043419648), and from an operator usability perspective we lose quite a bit of flexibility moving from a scripting language like PowerShell to a compiled language like C#.

We also need to start worrying about .NET versions. You'll find .NET v3.5 on a majority of Windows OS versions by default, but newer Windows 10 and Server 2016 systems will only have .NET v4.0+ installed by default. Another "gotcha" is that .NET is not enabled by default on all Windows OS versions either, you'll find that it needs to be explicitly enabled on Windows Server 2008 and earlier server OS versions. [SharpSploit](https://github.com/cobbr/SharpSploit) attempts to deal with this by targeting .NET v3.5 and v4.0 to get the most coverage possible, but you'll need to be careful to use the correct version on the correct system.

### To Console or Not to Console?

The most significant difference you will see between [SharpSploit](https://github.com/cobbr/SharpSploit) and most other offensive C# libraries that have been released so far, is that there is no `SharpSploit.exe`! [SharpSploit](https://github.com/cobbr/SharpSploit) is designed as a library, so there is only a `SharpSploit.dll`.

My intention is for [SharpSploit](https://github.com/cobbr/SharpSploit) to be primarily used as a library for operators to reference in their own toolsets. However, I anticipate some limitations from this implementation that will likely force me to add a console-based interface eventually. For instance, Cobalt Strike's `execute-assembly` module expects an application to have an EntryPoint (i.e. "main" function) to execute, so [SharpSploit](https://github.com/cobbr/SharpSploit) currently does not operate easily with Cobalt Strike. This is a great example of some of the flexibility issues with offensive C# we will have to solve in the transition from PowerShell.

Try not to worry about this too much for now, you'll see some other creative methods for [SharpSploit](https://github.com/cobbr/SharpSploit) execution from me here in the near future :) And I will likely add a follow-up post at some point on convenient methods for executing [SharpSploit](https://github.com/cobbr/SharpSploit) functions.

# SharpSploit

So what exactly does [SharpSploit](https://github.com/cobbr/SharpSploit) include? Let's dive in! `SharpSploit` currently includes 4 key high-level namespaces: `Credentials`, `Enumeration`, `Execution`, and `LateralMovement`. We'll walk through the details of each of these individually.

## SharpSploit.Credentials

The `SharpSploit.Credentials` namespace includes any classes that deal with...well, credentials! Currently, this includes all `Mimikatz` functionality as well as token manipulation.

### SharpSploit.Credentials.Mimikatz

The `SharpSploit.Credentials.Mimikatz` class implements the ability to execute any `Mimikatz` command. `SharpSploit`'s implementation uses an adapted version of [@subtee](https://twitter.com/subtee)'s PELoader and borrows from [@xorrior](https://twitter.com/xorrior)'s [implementation](https://github.com/xorrior/Random-CSharpTools/blob/master/DllLoader/DllLoader/PELoader.cs) as well to load [@gentilkiwi](https://twitter.com/gentilkiwi)'s excellent [Mimikatz](https://github.com/gentilkiwi/mimikatz) project.

I'd be remiss if I didn't also mention [@harmj0y](https://twitter.com/harmj0y)'s [SafetyKatz](https://github.com/GhostPack/SafetyKatz) project. The primary difference is that [SafetyKatz](https://github.com/GhostPack/SafetyKatz) provides convenience for using `Mimikatz` on a minidump of lsass.exe, while `SharpSploit.Credentials.Mimikatz` opens the user up to any `Mimikatz` command.

The following are the primary functions implemented in the `SharpSploit.Credentials.Mimikatz` class:

* `Command()` - Loads the Mimikatz PE with `PE.Load()` and executes a chosen Mimikatz command.
* `LogonPasswords()` - Loads the Mimikatz PE with `PE.Load()` and executes the Mimikatz command to retrieve plaintext passwords from LSASS. Equates to `Command("privilege::debug sekurlsa::logonPasswords")`. (Requires Admin)
* `SamDump()` - Loads the Mimikatz PE with `PE.Load()` and executes the Mimikatz command to retrieve password hashes from the SAM database. Equates to `Command("privilege::debug lsadump::sam")`. (Requires Admin)
* `LsaSecrets()` - Loads the Mimikatz PE with `PE.Load()` and executes the Mimikatz command to retrieve LSA secrets stored in registry. Equates to `Command("privilege::debug lsadump::secrets")`. (Requires Admin)
* `LsaCache()` - Loads the Mimikatz PE with `PE.Load()` and executes the Mimikatz command to retrieve Domain Cached Credentials hashes from registry. Equates to `Command("privilege::debug lsadump::cache")`. (Requires Admin)
* `Wdigest()` - Loads the Mimikatz PE with `PE.Load()` and executes the Mimikatz command to retrieve Wdigest credentials from registry. Equates to `Command("sekurlsa::wdigest")`.
* `All()` - Loads the Mimikatz PE with `PE.Load()` and executes each of the above builtin, local credential dumping commands. (Requires Admin)
* `DCSync()` - Loads the Mimikatz PE with `PE.Load()` and executes the "dcsync" module to retrieve the NTLM hash of a specified (or all) Domain user. (Requires Domain Admin (or equivalent rights))
* `PassTheHash()` - Loads the Mimikatz PE with `PE.Load()` and executes the "pth" module to start a new process as a user using an NTLM password hash for authentication. (Requires Admin)

### SharpSploit.Credentials.Tokens

The `SharpSploit.Credentials.Tokens` class implements token manipulation functions, as well as more complex actions that rely on token manipulation. While many token manipulation projects have already been created, `SharpSploit.Credentials.Tokens` often borrows from [@0xbadjuju](https://twitter.com/0xbadjuju)'s awesome [Tokenvator](https://github.com/0xbadjuju/Tokenvator) project.

The following are the primary functions implemented in the `SharpSploit.Credentials.Tokens` class:

* `WhoAmI()` - Gets the username of the currently used/impersonated token.
* `ImpersonateUser()` - Impersonate the token of a process owned by the specified user. Used to execute subsequent commands as the specified user. (Requires Admin)
* `ImpersonateProcess()` - Impersonate the token of the specified process. Used to execute subsequent commands as the user associated with the token of the specified process. (Requires Admin)
* `GetSystem()` - Impersonate the SYSTEM user. Equates to `ImpersonateUser("NT AUTHORITY\SYSTEM")`. (Requires Admin)
* `BypassUAC()` - Bypasses UAC through token duplication and spawns a specified process with high integrity. (Requires Admin)
* `RunAs()` - Makes a new token to run a specified function as a specified user with a specified password. Automatically calls `RevertToSelf()` after executing the function.
* `MakeToken()` - Makes a new token with a specified username and password, and impersonates it to conduct future actions as the specified user.
* `RevertToSelf()` - Ends the impersonation of any token, reverting back to the initial token associated with the current process. Useful in conjuction with functions that impersonate a token and do not automatically RevertToSelf, such as: `ImpersonateUser()`, `ImpersonateProcess()`, `GetSystem()`, and `MakeToken()`.
* `EnableTokenPrivilege()` - Enables a specified security privilege for a specified token. (Requires Admin)

## SharpSploit.Enumeration

The `SharpSploit.Enumeration` namespace includes any classes that do enumeration. Currently, this includes some basic local host-based enumeration, network enumeration (i.e. ping/port scanning), and Domain and Net enumeration. I think there's room for a lot of addition and improvement to this namespace, but it is a start.

### SharpSploit.Enumeration.Host

The `SharpSploit.Enumeration.Host` class does basic local host-based enumeration. This class is currently **very** basic. I plan to eventually make some useful additions, but currently there are many other more robust tools, such as [Seatbelt](https://github.com/GhostPack/Seatbelt).

The following are the primary functions implemented in the `SharpSploit.Enumeration.Host` class:

* `GetProcessList()` - Gets a list of running processes on the system.
* `CreateProcessDump()` - Creates a minidump of the memory of a running process. Useful for offline Mimikatz if dumping the LSASS process. (Requires Admin)
* `GetHostname()` - Gets the hostname of the system.
* `GetUsername()` - Gets the current Domain and username of the process running.
* `GetCurrentDirectory()` - Gets the current working directory full path.
* `GetDirectoryListing()` - Gets a directory listing of the current working directory.
* `ChangeCurrentDirectory()` - Changes the current directory by appending a specified string to the current working directory.
* `RegistryRead()` - Reads a value stored in registry.
* `RegistryWrite()` - Writes a value into the registry.

### SharpSploit.Enumeration.Network

The `SharpSploit.Enumeration.Network` class includes a threaded TCP ping/port scanner for network enumeration.

The following are the primary functions implemented in the `SharpSploit.Enumeration.Network` class:

* `PortScan()` - Conducts a port scan of specified computer(s) and port(s) and reports open ports.
* `Ping()` - Pings specified computer(s) to identify live systems.

### SharpSploit.Enumeration.Domain

The `SharpSploit.Enumeration.Domain` class provides libraries for Active Directory domain enumeration. This is a **partial** port of [@harmj0y](https://twitter.com/harmj0y)'s [PowerView](https://github.com/PowerShellMafia/PowerSploit/blob/dev/Recon/PowerView.ps1) project.

This class does not intend to be a direct or complete port of [PowerView](https://github.com/PowerShellMafia/PowerSploit/blob/dev/Recon/PowerView.ps1), things will be formatted and implemented differently. I should also mention [@0xthirteen](https://twitter.com/0xthirteen)'s [SharpView](https://github.com/tevora-threat/SharpView) project, which does aim to be a more direct/complete port of [PowerView](https://github.com/PowerShellMafia/PowerSploit/blob/dev/Recon/PowerView.ps1).

### SharpSploit.Enumeration.Domain.DomainSearcher

The `SharpSploit.Enumeration.Domain.DomainSearcher` class facilitates searching a particular AD domain with a set of credentials.

This is the first major difference you will notice when using `SharpSploit.Enumeration.Domain` compared to [PowerView](https://github.com/PowerShellMafia/PowerSploit/blob/dev/Recon/PowerView.ps1). [PowerView](https://github.com/PowerShellMafia/PowerSploit/blob/dev/Recon/PowerView.ps1) functions can all be executed independently as "static" functions, conforming to the nature of PowerShell as a scripting language. `SharpSploit` embraces the nature of an object-oriented programming language such as C#. You will need to create a `DomainSearcher` object that is setup to search a particular domain with a particular set of credentials, and call specific enumeration functions on that `DomainSearcher` object.

The `DomainSearcher` class contains the following primary functions:

* `GetDomainUsers()` - Gets a list of specified (or all) user `DomainObject`s in the current Domain.
* `GetDomainGroups()` - Gets a list of specified (or all) group `DomainObject`s in the current Domain.
* `GetDomainComputers()` - Gets a list of specified (or all) computer `DomainObject`s in the current Domain.
* `GetDomainSPNTickets()` - Gets `SPNTicket`s for specified `DomainObject`s.
* `Kerberoast()` - Gets a list of `SPNTicket`s for specified (or all) users with a SPN set in the current Domain.

### SharpSploit.Enumeration.Net

The `SharpSploit.Enumeration.Net` class continues the **partial** [PowerView](https://github.com/PowerShellMafia/PowerSploit/blob/dev/Recon/PowerView.ps1) port, and includes functions for enumerating local users, groups, logons, and sessions on remote computers by using Windows API functions (similar to the native, Windows "net.exe" utility). A key note is that the "WinNT" [PowerView](https://github.com/PowerShellMafia/PowerSploit/blob/dev/Recon/PowerView.ps1) collection method is not supported.

The `SharpSploit.Enumeration.Net` class includes the following primary functions:

* `GetNetLocalGroups()` - Gets a list of `LocalGroup`s from specified remote computer(s).
* `GetNetLocalGroupMembers()` - Gets a list of `LocalGroupMember`s from specified remote computer(s) for a specified group.
* `GetNetLoggedOnUsers()` - Gets a list of `LoggedOnUser`s from specified remote computer(s).
* `GetNetSessions()` - Gets a list of `SessionInfo`s from specified remote computer(s).

## SharpSploit.Execution

The `SharpSploit.Execution` namespace includes any classes that deal with executing "stuff". Currently, this includes PE loading, .NET assemblies, and shell/powershell commands.

### SharpSploit.Execution.Assembly

The `SharpSploit.Execution.Assembly` class includes methods for loading .NET assemblies and executing functions within the assembly through reflection.

The `SharpSploit.Execution.Assembly` class includes the following primary functions:

* `Load()` - Loads a .NET assembly byte array or base64-encoded byte array.
* `AssemblyExecute()` - Loads a .NET assembly byte array or base64-encoded byte array and executes a specified method within a specified type with specified parameters using reflection.

### SharpSploit.Execution.PE

The `SharpSploit.Execution.PE` class is planned to be a class for loading arbitrary PEs. Currently, it really only serves as a Mimikatz PE loader. This class is an adapted version of [@subtee](https://twitter.com/subtee)'s PE loader that has been made to support arbitrary PEs. As a warning, I have not been successful in loading any arbitrary PE with this class, so your mileage may vary. I hope that this will eventually be fully implemented.

The `SharpSploit.Execution.PE` class includes the following primary functions:

* `Load()` - Loads a PE with a specified byte array. (Requires Admin)
* `GetFunctionExport()` - Get a pointer to an exported function in a loaded PE. The pointer can then be used to execute the function in the PE.

### SharpSploit.Execution.Shell

The `SharpSploit.Execution.Shell` class includes methods for executing "shell" (`cmd.exe`) commands and PowerShell commands/scripts. It is possible to execute shell commands as an alternative user (provided that you have a valid password), similar to the native Windows `runas.exe` executable.

I'm particularly excited about the `PowerShellExecute()` function. The `PowerShellExecute()` function by default uses [@mattifestation](https://twitter.com/mattifestation)'s AMSI bypass and [@tifkin_](https://twitter.com/tifkin_)'s PowerShell logging bypass (which bypasses ScriptBlock logging and Module logging). An issue with executing some of these types of bypasses previously through PowerShell is that the bypasses themselves are logged or AMSI-scanned, creating a bit of a chicken and the egg problem. By utilizing C#, we can execute these bypasses in C# prior to the execution of any PowerShell, resulting in log-free and AMSI-free PowerShell execution!

The `SharpSploit.Execution.Shell` class includes the following primary functions: 

* `PowerShellExecute()` - Executes specified PowerShell code using the System.Management.Automation.dll and bypasses AMSI, ScriptBlock Logging, and Module Logging (but not Transcription Logging).
* `ShellExecute()` - Executes a specified Shell command, optionally with an alternative username and password. Equates to `ShellExecuteWithPath(ShellCommand, "C:\\WINDOWS\\System32")`
* `ShellExecuteWithPath()` - Executes a specified Shell command from a specified directory, optionally with an alternative username and password.

### SharpSploit.Execution.ShellCode

The `SharpSploit.Execution.ShellCode` class includes a method for executing shellcode. Shellcode execution is accomplished by copying it to pinned memory, modifying the memory permissions with `Win32.Kernel32.VirtualProtect()`, and executing with a .NET `delegate`. This class is based on code written by [@enigma0x3](https://twitter.com/enigma0x3).

The `SharpSploit.Execution.ShellCode` class includes the following primary function:

* `ShellCodeExecute()` - Executes a specified shellcode byte array by copying it to pinned memory, modifying the memory permissions with `Win32.Kernel32.VirtualProtect()`, and executing with a .NET `delegate`.

### SharpSploit.Execution.Win32

`SharpSploit.Execution.Win32` is a large library of PInvoke signatures and structures for Win32 API functions used by various `SharpSploit` functionality. [www.pinvoke.net](http://www.pinvoke.net) was an invaluable resource for the implementation of this class. I will not attempt to document all of the signatures here, but feel free to check them out in the `Win32.cs` file.

## SharpSploit.LateralMovement

The `SharpSploit.LateralMovement` namespace includes classes that allow for any type of code execution on remote computers. Currently, this only includes WMI lateral movement, but others will be coming soon!

### SharpSploit.LateralMovement.WMI

The `SharpSploit.LateralMovement.WMI` class includes basic WMI lateral movement capabilities. Currently, the only capability is to spawn a process on a remote system using the `Win32_Process.Create` WMI method with a specified username and password.

The `SharpSploit.LateralMovement.WMI` class currently includes only the following primary function:

* `WMIExecute()` - Execute a process on a remote system with Win32_Process Create with specified credentials.

## Testing and Documentation

Complete documentation for `SharpSploit` is available at [https://sharpsploit.cobbr.io/api/index.html](https://sharpsploit.cobbr.io/api/index.html)!

The `SharpSploit` project also includes `SharpSploit.Tests`, hooray for unit testing! By no means have I implemented complete coverage in these unit tests, but I am still working at it. I think it would be great if we as a security community could try to up our software development game a bit and include unit tests for offensive projects. They provide great examples for how to use our toolsets, promote code quality, and are useful for contributors.

## More To Do

This is just the beginning for `SharpSploit`! You'll notice I alluded to lots of needed additions to `SharpSploit` throughout the description above. I plan to continue working on and maintaining `SharpSploit`, so hopefully you will see lots of updates coming soon. The format of SharpSploit as a library is also very conducive to contribution from others. I am very open to pull requests and hope that others will want to contribute to the project to make it better for everyone.

## Defensive .NET

This is really an offensive post and meant as an intro to `SharpSploit`, but I will at least touch on the defensive side. After educating myself a bit, I'll plan on a more thorough defensive blog post as a follow up.

To begin discussing defense for .NET, it's important to clarify terms. I've used ".NET" and "C#" almost interchangably throughout this post, but there is a subtle distinction. The .NET framework is a library and runtime for Windows for "managed code". "Managed code" is any code that compiles to CIL (Common Intermediate Language, formerly referred to as "MSIL") and will be managed by the CLR (Common Language Runtime), which is the runtime for the .NET framework and converts CIL to "native" machine code. C# is an example of a language that uses managed code that compiles to CIL. Further, C# was created specifically to target .NET and it's object model aligns to the .NET object model. Other languages that can compile to CIL (and thus, can be executed under the .NET framework) include VB.NET and JScript.NET.

So while we've talked quite a bit about C# from the offensive side, from a defensive perspective what we really care about is .NET. Specifically, we care about the CIL code being interpreted by the CLR. The bad news is that traditionally, defenders do not have much optics into what is happening in .NET code. And even if they did, it may be hard to identify malicious .NET. In fact, that admission in [this tweet](https://twitter.com/blowdart/status/889458747516506113) from [@blowdart](https://twitter.com/blowdart) was what first got me excited about offensive .NET:

![Bad .NET Optics]({{site.baseurl}}/assets/images/bad-net-optics.png)

The good news is that this appears to be changing. I alluded to this at the beginning of the post, but it does seem as if .NET optics are in the works (discovered by [@mattifestation](https://twitter.com/mattifestation/status/1034197284043419648)):

![.NET Optics]({{site.baseurl}}/assets/images/net-optics.png)

However, as far as I know, there is no official guidance from Microsoft on how to utilize these optics quite yet. Though hopefully we will get some guidance and documentation as this appears to still be in the works.

Without these optics, what other options do we have? Well the good news is that there is still very few ways to execute .NET without touching disk at some point. And if you can recover the .NET assembly off of disk, it is very easy to reverse the assembly to source code and begin to identify key indicators (CIL code is still bytecode!).

The easiest and most popular way to execute malicious .NET assemblies is through the `System.Reflection.Assembly.Load()` method. So how will operators execute this method to kick off their malicious .NET assembly? Typically, either through `powershell.exe` or through scriptlet-based execution techniques such as `regsvr32.exe` and `wmic.exe` through techniques demonstrated by the great [DotNetToJScript](https://github.com/tyranid/DotNetToJScript) project by [@tiraniddo](https://twitter.com/tiraniddo). While those scriptlet-based techniques do have the ability to execute remotely hosted scriplets, this still results in the scriptlet hitting disk in a predictable location under `%LOCALAPPDATA%\Microsoft\Windows\Temporary Internet Files\` (which I'm not sure all red teamers have realized). This leaves `powershell.exe` as the last real way to execute .NET assemblies without touching disk, and PowerShell has great logging capabilities to ensure you capture the assembly in ScriptBlock logs! Once you have captured the assembly off disk or in logs, you will be able to easily reverse it to source and analyze using a .NET debugger, such as [dnSpy](https://github.com/0xd4d/dnSpy).

I hope this is a decent, if not brief, start to detecting malicious .NET. Again, once I further educate myself, I'll try to follow up with a more detailed post.

## Conclusion

This is just the beginning for `SharpSploit`! You'll notice I alluded to lots of needed additions to `SharpSploit` throughout this post. I plan to continue working on and maintaining `SharpSploit`, so hopefully you will see lots of updates coming soon.

I hope that others find `SharpSploit` helpful, particularly for those trying to diversify their toolset from PowerShell. And as I hinted to at the beginning of this post, `SharpSploit` is the first in a series of C# tooling that I have been developing over the last few months, so expect to see much more here soon!

## Credits

I owe a ton of credit to a lot of people. Nearly none of `SharpSploit` is truly original work. `SharpSploit` ports many modules written in PowerShell by others, utilizes techniques discovered by others, and borrows ideas and code from other C# projects as well. With that being said, I'd like to thank the following people for contributing to the project (whether they know they did or not *:)*):

* Justin Bui ([@youslydawg](https://twitter.com/youslydawg)) - For contributing the `SharpSploit.Enumeration.Host.CreateProcessDump()` function.
* Matt Graeber ([@mattifestation](https://twitter.com/mattifestation)), Will Schroeder ([@harmj0y](https://twitter.com/harmj0y)), and Ruben ([@FuzzySec](https://twitter.com/fuzzysec)) - For their work on [PowerSploit](https://github.com/PowerShellMafia/PowerSploit).
* Will Schroeder ([@harmj0y](https://twitter.com/harmj0y)) - For the [PowerView](https://github.com/PowerShellMafia/PowerSploit/blob/dev/Recon/PowerView.ps1) project.
* Alexander Leary ([@0xbadjuju](https://twitter.com/0xbadjuju)) - For the [Tokenvator](https://github.com/0xbadjuju/Tokenvator) project.
* James Foreshaw ([@tiraniddo](https://twitter.com/tiraniddo)) - For his discovery of the token duplication UAC bypass technique documented [here](https://tyranidslair.blogspot.com/2017/05/reading-your-way-around-uac-part-3.html).
* Matt Nelson ([@enigma0x3](https://twitter.com/enigma0x3)) - For his [Invoke-TokenDuplication](https://github.com/enigma0x3/Misc-PowerShell-Stuff/blob/master/Invoke-TokenDuplication.ps1) implementation of the token duplication UAC bypass, as well his C# shellcode execution method.
* Benjamin Delpy ([@gentilkiwi](https://twitter.com/gentilkiwi)) - For the [Mimikatz](https://github.com/gentilkiwi/mimikatz) project.
* Casey Smith ([@subtee](https://twitter.com/subtee)) - For his work on a C# PE Loader.
* Chris Ross ([@xorrior](https://twitter.com/xorrior)) - For his implementation of a Mimikatz PE Loader found [here](https://github.com/xorrior/Random-CSharpTools/blob/master/DllLoader/DllLoader/PELoader.cs).
* Matt Graeber ([@mattifestation](https://twitter.com/mattifestation)) - For discovery of the AMSI bypass found [here](https://twitter.com/mattifestation/status/735261120487772160).
* Lee Christensen ([@tifkin_](https://twitter.com/tifkin_)) - For the discovery of the PowerShell logging bypass found [here](https://github.com/leechristensen/Random/blob/master/CSharp/DisablePSLogging.cs).
* All the contributors to [www.pinvoke.net](www.pinvoke.net) - For numerous PInvoke signatures.