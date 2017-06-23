---
layout: post
title: "ObfuscatedEmpire - Updates and Pull Request!"
date: 2017-06-21 10:00:00 -0600
tags: ObfuscatedEmpire Invoke-Obfuscation Empire PowerShell
---

This post is intended to document the changes in ObfuscatedEmpire since [my initial introductory post to ObfuscatedEmpire]({{site.baseurl}}/ObfuscatedEmpire.html).

For details and motivations for ObfuscatedEmpire definitely start with that first post.

The first, most important change, is that ObfuscatedEmpire is now built on Empire 2.0! It wasn't until I pushed the final few changes to ObfuscatedEmpire based on Empire 1.6 that I realized it really should have been built on 2.0 all along. And unfortunately, I essentially had to rewrite the entire thing to accommodate the changes into Empire 2.0. But the good news is that the rewrite resulted in much cleaner code overall.

Aside from the upgrade to Empire 2.0, there isn't many functional differences in the new ObfuscatedEmpire. However, here is a short list of notable differences you should be aware of when using ObfuscatedEmpire:

* **Invoke-Obfuscation style syntax** - When specifying the optional global `obfuscate_command` or stager `ObfuscateCommand` options, you can now use fully compliant Invoke-Obfuscation style commands. In the initial release it was required to use the slightly malformed syntax `Token,String,1,home,Token,Command,1,home,Token,Variable,1` that works in the Linux version of PowerShell. ObfuscatedEmpire now translates the Invoke-Obfuscation compliant syntax to the usable Linux syntax so you can just specify the correct `Token\String\1,Token\Command\1,Token\Variable\1` obfuscation command syntax.
* **preobfuscate** - The `preobfuscate` command is now tab-completable! No more memorizing module paths! It's also important to note that `preobfuscate` obfuscates module_source files (located under {empire_root}/data/module_source), not modules themselves. (In Empire, a single module_source file can translate to many modules (i.e. PowerView)). And as always `preobfuscate all` will hit all of the source files.
* **Bug Fixes** - It turns out that obfuscation (particularly token obfuscation) is *difficult*! Obfuscation basically hits on every edge case you could think of in the PowerShell language. Many thanks to all of [Daniel Bohannon](https://twitter.com/danielhbohannon)'s work on Invoke-Obfuscation. I've worked a *lot* of obfuscation bug fixes into ObfuscatedEmpire, and have found that many of the Empire modules are now more stable when using the default `Token\All\1` obfuscation command.

### Moving Forward

On the subject of bug fixes, it is certainly still a work in progress and there are sure to be more issues that crop up. If you run into one *please* submit an issue!

If (read: When) you do run into obfuscation problems, remember that you do have full control over the obfuscation command! The default `Token\All\1` command is *heavy* obfuscation that has the potential to introduce issues. In particular, I've noticed that some of the PowerView modules have trouble with this heavy obfuscation. Please submit an issue whenever you find that obfuscation causes problems, but here is an alternate command to use that seems reliable and still uses fairly heavy obfuscation:

`Token\Comment\1,Token\Command\2,Token\Argument\2,Token\Member\4,Token\Variable\1,Token\Type\2`

And with the move to Empire 2.0 and all the bug fixes, ObfuscatedEmpire is finally in a state that is ready to be merged upstream to Empire! The pull request has been opened and can be tracked [here](https://github.com/EmpireProject/Empire/pull/585). It's a large change and will almost certainly take awhile to get merged in, but I am excited that Empire users will start to be introduced to the idea of using obfuscation in their workflow.
