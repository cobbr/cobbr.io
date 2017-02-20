---
layout: post
title: "ObfuscatedEmpire - Use an obfuscated, in-memory Powershell C2 channel to evade AV signatures"
date: 2017-2-19 12:00:00 -0600
tags: ObfuscatedEmpire Invoke-Obfuscation Empire AMSI Powershell
---
[ObfuscatedEmpire](https://github.com/) is an integration of two fantastic projects, [Invoke-Obfuscation](https://github.com/danielbohannon/Invoke-Obfuscation) and [Empire](https://github.com/EmpireProject/Empire). If you aren't already familiar with those projects, you really should go check them out first. But here's a quick summary for those who are unfamiliar:
* Empire is a Powershell post-exploitation agent. It's a powerful tool for attackers as it allows for a C2 channel to be run completely in-memory, without any malicious code touching disk, rendering traditional AV techniques ineffective.
* Invoke-Obfuscation is a Powershell script obfuscator. As the use of in-memory Powershell malware has grown, implementation of in-memory AV scanning of Powershell scripts has begun. Invoke-Obfuscation challenges all assumptions these in-memory Powershell AV signatures have been making.

## Motivations

ObfuscatedEmpire is a fork of Empire, with Invoke-Obfuscation baked directly into it's functionality. While nothing in ObfuscatedEmpire is "new", it does allow for something new: executing an **obfuscated** Powershell C2 channel totally in-memory.

While Empire is great for executing in-memory Powershell, it does little in the way of obfuscation. As we will see in a moment, this can leave behind some incriminating evidence in Window's EventLogs, and execution can even be blocked in-memory.

It seems Microsoft's answer to the new rage of using in-memory Powershell to evade AV, is **AMSI** (or Anti-Malware Scan Interface). AMSI is a simple API that allows *any* application to provide information to 3rd-party AV vendors. AV vendors develop signatures for that application, and block execution of anything they deem to be malicious. Microsoft utilizes AMSI for Powershell, indeed at the time of writing, I believe Powershell is the *only* app that utilizes AMSI.

Ultimately, AMSI can only be as effective as the signatures developed by these 3rd-party AV vendors. In the past, AV vendors have struggled to do exactly that, so I don't think it should surprise anyone that they haven't yet mastered AMSI signatures. And no tool highlights that fact more than Invoke-Obfuscation. Invoke-Obfuscation performs various types of obfuscation on Powershell scripts that fools these signatures. The exact obfuscation techniques used are outside the scope of this post, mainly because I don't really understand most of them. But I highly recommend watching one of Daniel Bohannon's [presentations](https://www.youtube.com/watch?v=6J8pw_bM-i4) on the subject.

From an attacker's perspective, Invoke-Obfuscation is great, but to properly use an obfuscated script you have to take a few steps:
1. Create and obfuscate some form of Powershell [remote-download cradle](https://gist.github.com/HarmJ0y/bb48307ffa663256e239).
2. Create an obfuscated Powershell one-liner that executes that remote download cradle.
3. Prepare and host an obfuscated version of the Powershell script you actually want to run. The remote download cradle will grab and execute this script.

This whole process has to be repeated for every script you want to run. This is where ObfuscatedEmpire comes in. By flipping a global 'obfuscate' switch on, ObfuscatedEmpire will perform server-side obfuscation of all stages of the C2 process. Let's take a look at some of the functionality.

## ObfuscatedEmpire - Usage

The first thing you'll want to take a look at is ObfuscatedEmpire's launcher menu. Launchers have added `Obfuscate` and `ObfuscateCommand` options. Simply set the `Obfuscate` flag to `True`, and you can optionally configure the Invoke-Obfuscation `ObfuscateCommand` to be used. If you have no idea what this should look like, go check out the documentation on the [Invoke-Obfuscation github page](https://github.com/danielbohannon/Invoke-Obfuscation).

![Launcher obfuscation options]({{site.baseurl}}/assets/images/launcher-obfuscate-options.png)

![Generated obfuscated launcher]({{site.baseurl}}/assets/images/obfuscated-launcher-generated.png)

Now you have an obfuscated Empire launcher. Before you go running that, you will also want to set the **global** `obfuscate` flag to `True`. And optionally set the **global** `obfuscate_command`. These control obfuscation for basically everything that is not a launcher. It obfuscates the entire agent-negotiation process as well as Empire's Powershell modules. Now, when you run that launcher, establish an agent, and execute a module, a dynamically-generated, obfuscated version of the module will be created and executed by the agent. Obfuscation can be a time-consuming process for lengthy scripts, so a global `preobfuscate` command is supplied to generate pre-obfuscated versions of **all** Empire modules. This allows you to obfuscate all of your modules prior to an engagement, and not have to wait on the obfuscation process in the middle of a project. You can also preobfuscate a selected Empire module, though this isn't demonstrated here.

![Preobfuscate Example]({{site.baseurl}}/assets/images/preobfuscate.png)

## ObfuscatedEmpire - A real-world example

Now that we know how it works, we are ready to see ObfuscatedEmpire used in a real-world scenario. Let's **assume breach** here, Empire is a *post-exploitation* agent after all. We have found some mechanism to execute code on a target machine, be it through a phishing scheme or capturing a domain user's credentials. For this example we will be using a fully updated Windows 10 machine with Windows Firewall and Windows Defender enabled. It's worth noting that not all AV solutions actually implement AMSI protection yet, in fact **most don't**. Some of the so-called "next-gen" endpoint protection solutions have implemented AMSI protection, along with Windows Defender. Without having access to a copy of any of those "next-gen" solutions for testing, I'll be using Windows Defender throughout this demonstration.

First, let's use no obfuscation options and see how our target system behaves. Generate your Empire launcher like normal and execute it using whatever code execution mechanism you have obtained.

![Unobfuscated Launcher]({{site.baseurl}}/assets/images/generate-unobfuscated-launcher-agent-started.png)

Let's check out what we can find on our target system. Here we see that the exact Powershell script we ran in-memory was logged and can be seen in EventViewer!

![Unobfuscated EventLog]({{site.baseurl}}/assets/images/unobfuscated-empire-eventlog.png)

We'll have to take a quick detour into Powershell logging to understand what is going on here.

### Powershell Logging - A Quick Detour

Powershell provides powerful logging capabilities of scripts executed on a system. Powershell has three different types of logging mechanisms: Module logging, Transcription, and ScriptBlock logging.

**Powershell Module logging**  logs an event for each *command* executed in a Powershell script. This can create an enormous amount of logs. FireEye [published a white paper](https://www.fireeye.com/content/dam/fireeye-www/global/en/solutions/pdfs/wp-lazanciyan-investigating-powershell-attacks.pdf) that demonstrated that running Invoke-Mimikatz alone generates 1200+ Powershell module logs. Module logging is **not** enabled by default on Windows 10.

**Transcription logging** is [described by Lee Holmes](https://blogs.msdn.microsoft.com/powershell/2015/06/09/powershell-the-blue-team/) as similar to what you would see if you were watching over someone's shoulder as they were typing the script. You see the script being executed exactly as it would be typed and you see the script's output. While at first this might seem to be exactly what a defender would need, all forms of obfuscation will persist to the transcription logs. A simple base64 encoding of a script makes the transcription logs a lot less useful. Transcription logging is also **not** enabled by default on Windows 10.

**ScriptBlock logging** is probably the most powerful form of logging available for Powershell. It logs Powershell scripts at the ScriptBlock level. A [ScriptBlock](https://msdn.microsoft.com/en-us/powershell/reference/4.0/microsoft.powershell.core/about/about_script_blocks) is "a collection of statements or expressions that can be used as a single unit". Where transcription logging logs scripts as they are **typed**, scriptblock logging logs the scriptblocks as they are **executed**, which makes a huge difference. This means that several layers of obfuscation are stripped off of the scriptblock prior to logging, such as the base64 encoding trick that hid us in transcription logging. Luckily, at least for us attackers, is that Invoke-Obfuscation provides **token-level** obfuscation which persists all the way to the ScriptBlock level. The code seen by AMSI is identical to what is available in the ScriptBlock logs. It is also important to note that ScriptBlock logging and AMSI are **enabled by default** on Windows 10.

### Back to hacking

Now that we have a better understanding of the Powershell logging capabilities, let's again take a look at the event logs generated by running an un-obfuscated Empire launcher on our target system.

![Unobfuscated EventLog]({{site.baseurl}}/assets/images/unobfuscated-empire-eventlog.png)

We can see that all of the Empire agent code can be found in the event logs. Also notice the `WARNING` level applied to these event logs. Interestingly, Windows Defender does not block execution of this unobfuscated code, but as an attacker we would still prefer to be stealthier and avoid those `WARNING` level event logs.

Now let's try to run everybody's favorite Empire module, `Invoke-Mimikatz`! Specifically, we will use the module `credentials/mimikatz/lsadump`, which will dump password hashes stored in lsass memory. While Mimikatz has the capability to dump *plaintext* passwords from lsass memory, since Windows 8.1 Microsoft has introduced the capability to disable the storing of plaintext credentials in memory by disabling the WDigest registry key. By default this registry is disabled, which means plaintext passwords will not be stored in memory. It is possible to re-enable this registry key, but let's assume that we are trying to remain stealthy and avoid altering registry keys.

Without enabling any of the obfuscation options in ObfuscatedEmpire, let's try it out.

![Mimikatz Fail]({{site.baseurl}}/assets/images/mimikatz-no-output.png)

Uh-oh, I'm not seeing any output! Let's see what happened on our Windows 10 target machine. We have an alert from Windows Defender!

![Windows Defender]({{site.baseurl}}/assets/images/defender-mimikatz.png)

At the bottom of that screenshot you can see that Windows Defender utilized AMSI to **block execution** of the `Invoke-Mimikatz` script. Apparently Microsoft did not get Benajmin Delpy's memo that [mimikatz is not a hacktool](http://blog.gentilkiwi.com/securite/mimikatz/hacktool-win32-mikatz-dll-hacktool-win64-mikatz-dll)!

Is this game over? Have the defenders won? Of course not, we've just moved the cat-and-mouse game of malware vs. AV signatures from disk to memory. Now let's use the obfuscation options in ObfuscatedEmpire to get some hashes.

First, let's kill off the agent that we were using, though it is worth noting that Windows Defender still did **not** stop the agent responsible for spawning off the blocked `Invoke-Mimikatz` script. But this time we want to use an obfuscated launcher.

![Obfuscated Launcher]({{site.baseurl}}/assets/images/obfuscated-launcher-generated-2.png)

Now set the global `obfuscate` switch to `True`, and execute the launcher.

![Obfuscated Agent]({{site.baseurl}}/assets/images/obfuscated-agent-started.png)

Nothing else needs to be done, all modules will be obfuscated by default. Execute the `credentials/mimikatz/lsadump` module and grab your hashes.

![Mimikatz lsadump]({{site.baseurl}}/assets/images/mimikatz-hash-output.png)
(Feel free to crack this hash, it uses everyone's favorite password-policy busting password scheme!)

This time we get no notifications from Windows Defender. We have bypassed the AV signatures through obfuscation! Let's take a look at those ScriptBlock logs to see why this works.

![Obfuscated Empire Eventlog]({{site.baseurl}}/assets/images/obfuscated-empire-eventlog.png)

![Obfuscated Mimikatz Eventlog]({{site.baseurl}}/assets/images/obfuscated-mimikatz-eventlog.png)

You can see the Invoke-Obfuscation **token-level** obfuscation persisted to the ScriptBlock logs. While I don't have access to Windows Defender's AV signatures, clearly they do not take the possibility of this token obfuscation into account. What's interesting is that these eventlogs are still being categorized under the `WARNING` label. I'm not sure how common this is to see in eventlogs with normal administrative Powershell functionality as added noise, but it's certainly something to investigate as defenders.

I should also emphasize the fact that all of this cool ScriptBlock logging and AMSI stuff only works on Powershell 5.0. So if you are anything like me, you are lazy and all of the above looks like an awful lot of work. In that case, just do `powershell -version 2` and be done with it!

## Conclusion

The AV vs. malware cat-and-mouse game has officially moved from disk to memory. Obfuscation poses a major problem for AV vendors that needs to be addressed sooner rather than later. ObfuscatedEmpire makes it easy for attackers to automatically utilize obfuscation techniques without needing to think too hard about it.

Long term, I'm hoping this obfuscation could be merged into Empire itself, but there could be a few obstacles to that since it requires using server-side Powershell on Linux, which is still in **alpha** state (and technically unsupported on Kali Linux), and makes substantial design changes to Empire module templates. I'll try to maintain ObfuscatedEmpire with the latest Empire commits until/if a merge is possible.

I would love to hear what people think about all of the stuff mentioned in this post. Feel free to drop a comment on this post or hit me up on twitter [@cobbr_io](https://twitter.com/cobbr_io) to communicate directly.

Grab the ObfuscatedEmpire code from [the github page](https://github.com/cobbr/ObfuscatedEmpire).

# Credits
* [@danielbohannon](https://twitter.com/danielbohannon) For the awesome Invoke-Obfuscation tool and for helping me understand how all of this Powershell logging/AMSI stuff works.
* Everyone who has worked on the [Empire](https://github.com/EmpireProject/Empire) project.
* The Microsoft Powershell team for the awesome Powershell logging capabilities, as well as making Powershell cross-platform. ObfuscatedEmpire wouldn't work without it.

## Additional Resources
More on Powershell logging:
* [https://blogs.msdn.microsoft.com/powershell/2015/06/09/powershell-the-blue-team/](https://blogs.msdn.microsoft.com/powershell/2015/06/09/powershell-the-blue-team/)
* [https://www.fireeye.com/blog/threat-research/2016/02/greater_visibilityt.html](https://www.fireeye.com/blog/threat-research/2016/02/greater_visibilityt.html)
* [https://www.fireeye.com/content/dam/fireeye-www/global/en/solutions/pdfs/wp-lazanciyan-investigating-powershell-attacks.pdf](https://www.fireeye.com/content/dam/fireeye-www/global/en/solutions/pdfs/wp-lazanciyan-investigating-powershell-attacks.pdf)

### Obligatory "don't do anything evil with this" warning

Don't do anything evil with this. Only use ObfuscatedEmpire on systems/networks you have permission to. Illegal use is not permitted.
