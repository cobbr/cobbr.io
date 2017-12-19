---
layout: post
title: "InsecurePowerShell - PowerShell without System.Management.Automation.dll"
date: 2017-12-18 10:00:00 -0600
tags: PowerShell InsecurePowerShell
---

PowerShell without System.Management.Automation.dll? I thought PowerShell *is* the System.Management.Automation.dll?

### PowerShell Without powershell.exe

A little bit of background: The title "PowerShell without System.Management.Automation.dll" derives from a lot of research that has been done about "PowerShell without `powershell.exe`". The idea being, that `powershell.exe` is just a process that *hosts* the System.Management.Automation.dll. And at it's core, that's what PowerShell really is, the System.Management.Automation.dll. There are other native Windows processes that are "PowerShell hosts" as well, including `powershell_ise.exe`.

However, we can also just create our own process that hosts the System.Management.Automation.dll, as several open-source projects have aimed to do, including [UnmanagedPowerShell](https://github.com/leechristensen/UnmanagedPowerShell), [SharpPick](https://github.com/PowerShellEmpire/PowerTools/tree/master/PowerPick/SharpPick), [PSAttack](https://github.com/jaredhaight/PSAttack), and [nps](https://github.com/Ben0xA/nps).

Of course, one of the benefits of using PowerShell in the first place is that `powershell.exe` is a Microsoft-signed binary that can help us get around Application Whitelisting solutions. So we lose the Application Whitelisting bypass benefit, but we are able to execute PowerShell in cases that `powershell.exe` is being blocked.

### PowerShell Without System.Management.Automation.dll

[InsecurePowerShell](https://github.com/cobbr/InsecurePowerShell) takes this one step further. If we are going to create a new "PowerShell host" binary and drop it to disk anyway, why not host a **modified version** of the System.Management.Automation.dll? As (hopefully) everyone knows by now, PowerShell has much improved security features, particularly in v5, including: ScriptBlock Logging, Module Logging, Transcription Logging, AMSI introspection, Constrained Language Mode, etc. All of these security features are implemented in the System.Management.Automation.dll, so how about we host a more fun version of the System.Management.Automation.dll that has all the features we want from the newest versions of PowerShell but none of the protections? This is what InsecurePowerShell aims to do. So what I really mean by PowerShell "without" System.Management.Automation.dll is PowerShell without the **default** System.Management.Automation.dll.

InsecurePowerShell is a fork of the [open-source PowerShell Core v6.0.0](https://github.com/PowerShell/PowerShell) with just a few modifications. InsecurePowerShell removes the following security features from PowerShell:
* AMSI - InsecurePowerShell does not submit any PowerShell code to the AMSI, even when there is an actively listening AntiMalware Provider.
* PowerShell Logging - InsecurePowerShell disables ScriptBlockLogging, Module Logging, and Transcription Logging. Even if they are enabled in Group Policy, these settings are ignored.
* LanguageModes - InsecurePowerShell always runs PowerShell code in `FullLanguage` mode. Attempting to set InsecurePowerShell to alternative LanguageModes, such as `ConstrainedLanguage` mode or `RestrictedLanguage` mode does not take any affect.
* ETW - InsecurePowerShell does not utilize ETW (Event Tracing for Windows).

## Usage

InsecurePowerShell is built exactly the same way as PowerShell Core and can be used exactly the same way. The [release](https://github.com/cobbr/InsecurePowerShell/releases/) of InsecurePowerShell includes a binary named `pwsh.exe`. This can be used exactly the same way as the standard PowerShell Core. It can be used interactively or non-interactively with the `-Command` or `-EncodedCommand` parameters. For example:

```
PS C:\InsecurePowerShell\> .\pwsh.exe
PowerShell v6.0.0-rc.2-67-g642a8fe0eb0b49f4046e434dc16748ea5c963d51
Copyright (c) Microsoft Corporation. All rights reserved.

https://aka.ms/pscore6-docs
Type 'help' to get help.

PS C:\InsecurePowerShell\> $Execution.SessionState.LanguageMode = 'ConstrainedLanguage'
PS C:\InsecurePowerShell\> $Execution.SessionState.LanguageMode
FullLanguage
PS C:\InsecurePowerShell\> Get-WinEvent -FilterHashtable @{ProviderName="Microsoft-Windows-PowerShell"; Id=4104} | % Message
Creating Scriptblock text (1 of 1):
.\pwsh.exe

ScriptBlock ID: 877d94f3-4bb5-4a26-88e3-58bb8091e1d8
Path:
Creating Scriptblock text (1 of 1):
prompt

ScriptBlock ID: 72983606-2c4d-4266-808c-280c718550c4
Path:
```

### InsecurePowerShellHost

In addition to this near `pwsh.exe` clone, I've created a .NET Core application called [InsecurePowerShellHost](https://github.com/cobbr/InsecurePowerShellHost). Just like it sounds, InsecurePowerShellHost is an application that hosts the modified `System.Management.Automation.dll` created with InsecurePowerShell. InsecurePowerShellHost can only be used non-interactively with the `--Command` and `--EncodedCommand` parameters. The main benefit of using InsecurePowerShellHost over InsecurePowerShell is that it is a bit lighterweight than the full PowerShell Core build included in InsecurePowerShell.

An example of InsecurePowerShellHost usage:

```
PS C:\InsecurePowerShellHost\> .\InsecurePowerShellHost.exe
usage: InsecurePowerShellHost.exe [--EncodedCommand encoded_command | --Command command]
PS C:\InsecurePowerShellHost\> .\InsecurePowerShellHost.exe --Command "`$Execution.SessionState.LanguageMode = 'ConstrainedLanguage'; `$Execution.SessionState.LanguageMode"
FullLanguage
```

## Conclusion

InsecurePowerShell/InsecurePowerShellHost are really simple concepts. This isn't anything groundbreaking. There are also plenty of downsides that could prevent this from being a real practical technique for most red teamers. But I wanted to demonstrate the point that we can create custom versions of PowerShell that do not employ the same security features in environments without a strong Application Whitelisting solution, and I think InsecurePowerShell demonstrates that point well.

With that in mind, let's talk about some of the practical pros and cons of using InsecurePowerShell for red teamers.

## Pros and Cons

### Pros

* **PowerShell Without the Security** - The biggest benefit by far, and the real motivation for InsecurePowerShell is to be able to execute PowerShell without the security features: no AMSI, no ScriptBlock Logging, no Module Logging, no Transcription Logging, etc.
* **Compatibility** - Being a .NET Core application, we can execute InsecurePowerShell on Windows 7 SP1, Windows 8.1, Windows 10, Windows Server 2008 R2 SP1, Windows Server 2012, and Windows Server 2016.
* **Features of PowerShell Core v6.0** - We get all of the functionality of PowerShell Core v6.0. As red teamers, we are often limited to the functionality of PowerShell v2.0 in order to avoid the newer PowerShell security features. No longer!

### Cons

* **Touching Disk** - As a .NET Core application, not only do we have to drop a binary to disk, but we also need to drop all the supporting DLLs for .NET Core as well. These will mostly be Microsoft-signed binaries, however, we would always prefer to touch disk the least amount possible. InsecurePowerShell really does **not** conform to the "living off the land" principle at all.
* **Application Whitelisting** - InsecurePowerShell is not going to work with a strong Application Whitelisting solution. We lose one of the big original benefits of `powershell.exe`.
* **PowerShell Core != Windows PowerShell** - InsecurePowerShell is a fork of PowerShell Core, **not** Windows PowerShell. There is not feature parity between Windows PowerShell and PowerShell Core. This might be the biggest practical concern with using InsecurePowerShell, pre-existing PowerShell tooling may or may not work correctly in PowerShell Core.

## Defense

InsecurePowerShell/InsecurePowerShellHost are just binaries/DLLs that need to be dropped to disk. Defense could be as easy as blacklisting the modified System.Management.Automation.dll. And you might as well blacklist older versions of System.Management.Automation.dll, such as v2.0, while you are it!

Blacklisting is a quick, simple fix, but the real answer is an Application Whitelisting solution. Defeating a blacklist could be as simple as recompiling InsecurePowerShell/InsecurePowerShellHost.

## More To Do

InsecurePowershell is a simple concept that I really don't plan on actively maintaining. However, there are a couple of interesting items that could enhance InsecurePowerShell that I may revisit in the future:

* **Dynamically Load Assemblies** - 99% of the time I spent working on InsecurePowershell was spent trying to create a single binary that dynamically loads the modified System.Management.Automation.dll and the .NET Core DLLs so that InsecurePowershell could be distributed as a single binary file, however this turned out to be more difficult than I anticipated. So for now, InsecurePowerShell exists as a single ZIP archive that must be extracted out to many files. However, I may revisit this in the future.
* **PowerShell Core Agent** - It might be convenient to have a full PowerShell C2 agent that is compatible with PowerShell Core. The lack of feature parity between WindowsPowerShell and PowerShell Core will probably cause most pre-existing PowerShell C2 agents to fail.

---

InsecurePowerShell can be found [here](https://github.com/cobbr/InsecurePowerShell) and InsecurePowerShellHost can be found [here](https://github.com/cobbr/InsecurePowerShellHost).