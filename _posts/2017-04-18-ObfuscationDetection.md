---
layout: post
title: "Trying to Detect PowerShell Obfuscation Through Character Frequency"
date: 2017-4-18 09:00:00 -0600
tags: ObfuscatedEmpire Invoke-ObfuscationDetection Invoke-Obfuscation AMSI PowerShell
---
[In my last post]({{site.baseurl}}/ObfuscatedEmpire.html) describing the usage of ObfuscatedEmpire for automating PowerShell obfuscation within a C2 channel, I mentioned a technique others have proposed for detecting malicious PowerShell scripts. The technique, originally suggested by Lee Holmes (of Microsoft), is to search for signs of obfuscation itself.

For example, a token obfuscation trick utilized by [Invoke-Obfuscation](https://github.com/danielbohannon/Invoke-Obfuscation) is to insert apostrophes into function names and other tokens. `Invoke-Empire` might become ```iN`v`OK`e-`eM`p`IR`e```. These are functionally equivalent within PowerShell, but will break AV signatures matching the literal string "Invoke-Empire". But should we really expect that half of the characters in a script consist of apostrophe characters? Lee wrote about these types of detection methods as early as November 2015 [here](https://www.leeholmes.com/blog/2015/11/13/detecting-obfuscated-powershell/). In this post, however, I'll mainly be referencing his more recent article that was written partially as a reaction to Invoke-Obfuscation, which you can read [here](https://www.leeholmes.com/blog/2016/10/22/more-detecting-obfuscated-powershell/). He provides us with some really awesome PowerShell functions that actually begin to implement these obfuscation detection techniques, `Measure-CharacterFrequency` and `Measure-VectorSimilarity`.

This post is essentially just me trying to reproduce the results found by Lee in his article and provide a wrapper script for his work that can be used to detect obfuscated scripts.

This script, [Invoke-ObfuscationDetection](https://gist.github.com/cobbr/acbe5cc7a186726d4e309070187beee6), serves as a wrapper for Lee's functions that could be used to operationalize character analysis based obfuscation detection. `Invoke-ObfuscationDetection` defines a baseline of the "normal" character distribution of PowerShell scripts, calculates the character distribution of the given PowerShell script, defines a vector similarity of the character distribution the given script must meet, and then returns a boolean `True` or `False` answer to whether the script is obfuscated or not based on the result.

I began writing `Invoke-ObfuscationDetection` and this article hoping to demonstrate that this technique could serve as an operationally effective means of detecting or even **blocking** (through AMSI) obfuscated PowerShell scripts. After playing around with it a bit, I don't believe the detection is effective enough to actually block scripts (we'll see why later), but it could be useful to detect potentially obfuscated scripts that are worthy of further investigation.

## Usage

`Invoke-ObfuscationDetection` returns a boolean "IsObfuscated" result, given a string containing a script:
```
PS> Invoke-ObfuscationDetection -Script 'iN`v`OK`e-`eM`p`IR`e'

Obfuscated
----------
      True
```

`Invoke-ObfuscationDetection` also accepts filenames containing scripts as input with the `-ScriptPath` parameter. We can also take values from the pipeline:
```
PS /opt/ObfuscatedEmpire/data/obfuscated_module_source/> Get-ChildItem -Recurse -Include *.ps1 | Invoke-ObfuscationDetection | % { $_.Obfuscated } | Group-Object

Count Name                      Group
----- ----                      -----
   72 True                      {True, True, True, True...}
    2 False                     {False, False}
```

That command shows the results of `Invoke-ObfuscationDetection` on Empire modules obfuscated by `Invoke-Obfuscation`'s `Token\All\1` command. Pretty effective!

We can also feed ScriptBlock logs through `Invoke-ObfuscationDetection` (Enable ScriptBlock logging!):
```
PS> Get-WinEvent -FilterHashtable @{ProviderName="Microsoft-Windows-PowerShell"; Id = 4104} | % { [PSCustomObject] @{ ScriptName = $_.Properties[3].Value; Script = $_.Properties[2].Value } } | Invoke-ObfuscationDetection | Select -First 2
Name                                 Obfuscated
----                                 ----------
2980cef2-ed31-4146-870a-a395b2d3debf       True
431be04f-98a5-47cf-8e47-e565ccf6e520      False
```

## Methodology

Before discussing the effectiveness of `Invoke-ObfuscationDetection`, I think it's important to explain the implementation and testing methodology used, because it certainly is not robust. One of the challenges in the implementation of `Invoke-ObfuscationDetection` was determining what constitutes "normal" PowerShell scripts. The way I accomplished this was by downloading every script on [poshcode.org](http://poshcode.org), removing the intentionally obfuscated scripts and those identified as malware by Windows Defender (leaving a total of 5552 scripts), and using the `Measure-CharacterFrequency` function (on half of them) to get the average character distribution.

The second challenge is determining the acceptable difference from the average character distribution. Not every script is going to totally conform to the average character distribution. We will use the `Measure-VectorSimilarity` function to measure the difference from the average character distribution. But how do we decide the acceptable difference? I found it easiest to use an empirical approach. We take a sample of half of our 5552 scripts to train and half to test (to avoid overfitting). The first half is what we feed to `Measure-CharacterFrequency` to determine average character distribution. The second half of scripts will be our testing set.

## Effectiveness

There are two important things to test when trying to implement an AV function (which is essentially what we are doing), the false positive rate and false negative rate. A false negative occurs when we feed an obfuscated script to `Invoke-ObfuscationDetection`, but it is not detected as obfuscated. A false positive occurs when we feed an unobfuscated script to `Invoke-ObfuscationDetection`, but it is detected as obfuscated. We would like to minimize both of these numbers. Decreasing either rate increases the other, the key is to find a balance that provides acceptable false negative and false positive rates.

In our case the variable that determines the false positive/negative rates is the vector similarity requirement we choose. We will use our testing set of scripts to empirically determine the optimal vector similarity requirement to minimize the false positive and negative rates, but of course, we can't minimize both. We take our training set of scripts and obfuscate them using `Invoke-Obfuscation`'s `Token\All\1` command. We submit all of the unobfuscated versions to `Invoke-ObfuscationDetection` to determine the false positive rate, and submit all of the obfuscated versions to `Invoke-ObfuscationDetection` to determine the false negative. 

We will run this experiment with various vector similarity requirements, and compare false positive/negative rates at each of these requirements. The following chart helps to visualize the data (X-axis is similarity requirement, Y-axis is false positive/negative percentage):

![Rates Chart]({{site.baseurl}}/assets/images/rates-chart.png)

And the raw data, for some extra details:

![Rates Table]({{site.baseurl}}/assets/images/rates-table.png)

Lee suggested a vector similarity requirement of 0.8, and our data shows similar results. In his article, he mentions that about 2% of scripts fall lower than a 0.8 vector similarity rate, and we see similar results ourselves. With this 0.8 requirement we have about an **8% false negative rate**, which seems reasonable. Unfortunately, the **2% false positive rate** is probably too high to actually prevent execution using AMSI. Imagine if your AV flagged 1 in every 50 files on your computer as malware, you would never finish clearing the notifications! We also can't raise the 0.8 vector similarity requirement much higher. Any higher and the false positive rate starts to skyrocket. However, I do think it is reasonable to flag 2% of PowerShell scripts as potentially obfuscated for them to be further investigated by humans.

`Invoke-ObfuscationDetection` enforces this vector similarity rate of 0.8 by default. You can also specify another more strict or lenient requirement depending on if you want to make more or less work for yourself:

```
PS> Invoke-ObfuscationDetection -Script "Inv`ok`e-Ex`pre`s`s`ion 'Write-Host test'"

Obfuscated
----------
      False
PS> Invoke-ObfuscationDetection -Script "Inv`ok`e-Ex`pre`s`s`ion 'Write-Host test'" -RequiredSimilarity 0.85

Obfuscated
----------
      True
```

## Minimizing Obfuscation

The real problem here is that the obfuscation mechanism used (`Invoke-Obfuscation`'s `Token\All\1` command) is heavy, randomized obfuscation of all tokens in a script. Any attempt at more subtle obfuscation makes the detection metrics much worse. And unfortunately, the majority of AMSI signatures I have seen don't require much obfuscation to successfully evade AV detection.

Let's check out what our false positive/negative rates are after **slightly** less obfuscation using `Invoke-Obfuscation`'s `Token\String\1` command (X-axis is similarity requirement, Y-axis is false positive/negative percentage):

![Rates Chart]({{site.baseurl}}/assets/images/rates-chart-2.png)

And again the raw data, for the curious:

![Rates Table]({{site.baseurl}}/assets/images/rates-table-2.png)

Uh oh. Suddenly our optimal 0.8 similarity requirement results in an **80% false negative rate**! I'm not sure how well attackers are currently minimizing their obfuscation out in the wild, but it's something we need to be aware of. Character analysis should definitely be useful to detect some more obvious obfuscation, but it's also not something we can rely on.

I'm currently working on a new project that begins to head in the direction of **minimal** obfuscation to further demonstrate to defenders the potential for abuse. If we begin to rely on ineffective or incomplete obfuscation detection to alert us to malicious PowerShell obfuscation, we will miss that middle area where scripts are obfuscated enough to pass AV detection but not enough to be detected through character analysis.

Much more on that topic to come.

# Credits

I borrowed heavily from Lee Holmes in this post. Here are his posts I referenced:

* His first article detailing PowerShell obfuscation detection: [https://www.leeholmes.com/blog/2015/11/13/detecting-obfuscated-powershell/](https://www.leeholmes.com/blog/2015/11/13/detecting-obfuscated-powershell/)
* The follow-up that is referenced most in this article: [https://www.leeholmes.com/blog/2016/10/22/more-detecting-obfuscated-powershell/](https://www.leeholmes.com/blog/2016/10/22/more-detecting-obfuscated-powershell/)

An additional, relevant post for defenders:
* [https://blogs.msdn.microsoft.com/powershell/2015/06/09/powershell-the-blue-team/](https://blogs.msdn.microsoft.com/powershell/2015/06/09/powershell-the-blue-team/)