---
layout: page
title: About
sidebar_link: true
permalink: /about/
---

I work as a red teamer and Operator at [SpecterOps](https://specterops.io/). I enjoy security-related software development in my spare time, and tend to write about topics related to offensive PowerShell, .NET, and Windows security in general.

I intend for this blog to be a platform to document and share the things I am working on or interest me. I don't claim to be an expert.

### Projects

Projects I am currently working on:

* **Covenant** - [Covenant](https://github.com/cobbr/Covenant) is a .NET command and control framework that aims to highlight the attack surface of .NET, make the use of offensive .NET tradecraft easier, and serve as a collaborative command and control platform for red teamers.
* **SharpSploit** - [SharpSploit](https://github.com/cobbr/SharpSploit) is a .NET post-exploitation library written in C# that aims to highlight the attack surface of .NET and make the use of offensive .NET easier for red teamers.
* **PSAmsi** - [PSAmsi](https://github.com/cobbr/PSAmsi) is a tool for auditing and defeating AMSI signatures.

Projects I help to maintain:

* **Invoke-Obfuscation** - I help to maintain [Invoke-Obfuscation](https://github.com/danielbohannon/Invoke-Obfuscation), a PowerShell obfuscator, written by [Daniel Bohannon](https://twitter.com/danielhbohannon).

Projects I have worked on in the past:

* **ObfuscatedEmpire** - [ObfuscatedEmpire](https://github.com/cobbr/ObfuscatedEmpire) is an integration of [Empire](https://github.com/EmpireProject/Empire) and [Invoke-Obfuscation](https://github.com/danielbohannon/Invoke-Obfuscation), for automating obfuscation within a PowerShell C2 channel. Introductory blog post and "how to" available [here]({{site.baseurl}}/ObfuscatedEmpire.html). **This has been merged upstream to [Empire](https://github.com/EmpireProject/Empire), so just use that now!**

### Presentations

Presentations I have given at conferences:

* **BSides DFW 2017** - "PSAmsi: Offensive PowerShell Interaction with the AMSI" - A deeper dive into how the Anti-Malware Scan Interface (AMSI) works, how to enumerate AMSI signatures using PSAmsi, and how to simultaneously defeat AMSI signatures and obfuscation detection by minimizing obfuscation using PSAmsi. Also introduced the concept of AbstractSyntaxTree-based PowerShell obfuscation as a stealthier form of obfuscation. Included demos of [PSAmsi](https://github.com/cobbr/PSAmsi), a tool for auditing and defeating AMSI signatures. [Slides available here](https://github.com/cobbr/slides/blob/master/BSides/DFW/PSAmsi%20-%20Offensive%20PowerShell%20Interaction%20with%20the%20AMSI.pdf)

* **DerbyCon 2017** - "PSAmsi: Offensive PowerShell Interaction with the AMSI" - A deeper dive into how the Anti-Malware Scan Interface (AMSI) works, how to enumerate AMSI signatures using PSAmsi, and how to simultaneously defeat AMSI signatures and obfuscation detection by minimizing obfuscation using PSAmsi. Also demonstrated how to use PSAmsi generated payloads within Empire. Included demos of [PSAmsi](https://github.com/cobbr/PSAmsi), a tool for auditing and defeating AMSI signatures. [Slides available here](https://github.com/cobbr/slides/blob/master/DerbyCon/PSAmsi%20-%20Offensive%20PowerShell%20Interaction%20with%20the%20AMSI.pdf), [Recording available here](https://www.youtube.com/watch?v=rEFyalXfQWk)

* **BSides Austin 2017** - "Obfuscating The Empire" - An introduction to PowerShell logging capabilities, the Anti-Malware Scan Interface (AMSI) introduced in Windows 10, and how to use obfuscation to evade AV signatures. Included demos of [ObfuscatedEmpire](https://github.com/cobbr/ObfuscatedEmpire), an integration of [Invoke-Obfuscation](https://github.com/danielbohannon/Invoke-Obfuscation) and [Empire](https://github.com/EmpireProject/Empire). [Slides available here](https://github.com/cobbr/slides/blob/master/BSides/Austin/Obfuscating%20The%20Empire.pdf).

