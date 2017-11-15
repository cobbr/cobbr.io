---
layout: page
title: About
permalink: /about/
---

I work as a pentester and consultant at [Protiviti](https://www.protiviti.com/).

I intend for this blog to be a platform to document and share the things I am working on or interest me. I don't claim to be an expert.

Projects I am currently working on:

* [PSAmsi](https://github.com/cobbr/PSAmsi) - A tool for auditing and defeating AMSI signatures.
* [Invoke-Obfuscation](https://github.com/danielbohannon/Invoke-Obfuscation) - I help to maintain [Invoke-Obfuscation](https://github.com/danielbohannon/Invoke-Obfuscation), a PowerShell obfuscator, written by [Daniel Bohannon](https://twitter.com/danielhbohannon).

Projects I have worked on in the past:

* [ObfuscatedEmpire](https://github.com/cobbr/ObfuscatedEmpire) - An integration of [Empire](https://github.com/EmpireProject/Empire) and [Invoke-Obfuscation](https://github.com/danielbohannon/Invoke-Obfuscation), for automating obfuscation within a PowerShell C2 channel. Introductory blog post and full explanation available [here]({{site.baseurl}}/ObfuscatedEmpire.html). **This has been merge upstream to [Empire](https://github.com/EmpireProject/Empire), so just use that!**

Presentations:

* **"PSAmsi: Offensive PowerShell Interaction with the AMSI" - BSides DFW 2017** - A deeper dive into how the Anti-Malware Scan Interface (AMSI) works, how to enumerate AMSI signatures using PSAmsi, and how to simultaneously defeat AMSI signatures and obfuscation detection by minimizing obfuscation using PSAmsi. Also introduced the concept of AbstractSyntaxTree-based PowerShell obfuscation as a stealthier form of obfuscation. Included demos of [PSAmsi](https://github.com/cobbr/PSAmsi), a tool for auditing and defeating AMSI signatures. [Slides available here](https://github.com/cobbr/slides/blob/master/BSides/DFW/PSAmsi%20-%20Offensive%20PowerShell%20Interaction%20with%20the%20AMSI.pdf)

* **"PSAmsi: Offensive PowerShell Interaction with the AMSI" - DerbyCon 2017** - A deeper dive into how the Anti-Malware Scan Interface (AMSI) works, how to enumerate AMSI signatures using PSAmsi, and how to simultaneously defeat AMSI signatures and obfuscation detection by minimizing obfuscation using PSAmsi. Also demonstrated how to use PSAmsi generated payloads within Empire. Included demos of [PSAmsi](https://github.com/cobbr/PSAmsi), a tool for auditing and defeating AMSI signatures. [Slides available here](https://github.com/cobbr/slides/blob/master/DerbyCon/PSAmsi%20-%20Offensive%20PowerShell%20Interaction%20with%20the%20AMSI.pdf) [Recording available here](https://www.youtube.com/watch?v=rEFyalXfQWk)

* **"Obfuscating The Empire" - BSides Austin 2017** - An introduction to PowerShell logging capabilities, the Anti-Malware Scan Interface (AMSI) introduced in Windows 10, and how to use obfuscation to evade AV signatures. Included demos of [ObfuscatedEmpire](https://github.com/cobbr/ObfuscatedEmpire), an integration of [Invoke-Obfuscation](https://github.com/danielbohannon/Invoke-Obfuscation) and [Empire](https://github.com/EmpireProject/Empire). [Slides available here](https://github.com/cobbr/slides/blob/master/BSides/Austin/Obfuscating%20The%20Empire.pdf).
