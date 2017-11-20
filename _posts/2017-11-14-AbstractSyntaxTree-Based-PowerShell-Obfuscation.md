---
layout: post
title: "AbstractSyntaxTree-Based PowerShell Obfuscation"
date: 2017-11-20 12:00:00 -0600
tags: PSAmsi PowerShell AbstractSyntaxTree
---

The biggest addition made to PSAmsi in the recent release of [PSAmsi v1.1](https://github.com/cobbr/PSAmsi) was AbstractSyntaxTree-based PowerShell "obfuscation". I use "obfuscation" in quotes here, and hopefully you'll see why I do by the end of this post.

## AbstractSyntaxTree?

So what is an AbstractSyntaxTree? An AbstractSyntaxTree ("AST") is a commonly used structure to represent and parse source code in both compiled and interepreted languages. PowerShell includes a built-in AST since PowerShell v3. PowerShell is unique in that it exposes the AST structure in a way that is friendly to developers and is [documented extensively](https://docs.microsoft.com/en-us/dotnet/api/system.management.automation.language).

The following is an example of a full AbstractSyntaxTree represenation of a PowerShell script:

![Example AST]({{site.baseurl}}/assets/images/example-ast.png)


## Example

We'll be using the following second example script throughout the remainder of this post:

![Test-AstObfuscation Function]({{site.baseurl}}/assets/images/Test-AstObfuscation.png)

This is just a stupid PowerShell function that does nothing useful, but will help demonstrate some of the AST-based obfuscation techniques.

### PSToken Based Obfuscation

To help understand the benefits of AbstractSyntaxTree-based obfuscation, I think it's helpful to first understand how PSToken based obfuscation works. Tokens or "PSTokens" are another syntactical structure for parsing and representing PowerShell code, that have been used since PowerShell v2. A PowerShell script is essential made up of a big list of PSTokens, often delimited with whitespace. Where the AST represents the script in a more complex structure that is vaguely grouped in functional components, PSTokens are a simpler list.

In PSAmsi v1.0, the only obfuscation used was [Invoke-Obfuscation](https://github.com/danielbohannon/Invoke-Obfuscation)'s "Token" obfuscation, which does PSToken-based obfuscation. Invoke-Obfuscation has a library of options for obfuscation of various types of PSTokens. At a high-level, it essentially iterates over all of the PSTokens within a script, obfuscates each one in isolation, and combines the obfuscated pieces back together at the end.

For example, PSToken obfuscation for our example script would occur, somewhat, like this:

We begin iterating over the PSTokens, the first one being a CommandArgument token. We know we can insert tick characters into CommandArgument tokens, so we do that:

```
Type            Content                     ObfuscatedContent
----            -------                     -----------------
CommandArgument Test-AstObfuscation   ->    TE`s`t-AS`TOBFus`CAtIon
```

Next we have a `ParameterSetName` Member token, where we can't insert ticks. So we just randomly case the characters:

```
Type            Content                     ObfuscatedContent
----            -------                     -----------------
CommandArgument Test-AstObfuscation   ->    TE`s`t-AS`TOBFus`CAtIon
Member          ParameterSetName      ->    ParamEterseTNAME
```

Next we have a String token, where we have a few options, but we'll just add ticks here:

```
Type            Content                     ObfuscatedContent
----            -------                     -----------------
CommandArgument Test-AstObfuscation   ->    TE`s`t-AS`TOBFus`CAtIon
Member          ParameterSetName      ->    ParamEterseTNAME
String          Set1                  ->    “S`et1”
```

This iterative process continues throughout the entire script

```
Type            Content                                  ObfuscatedContent
----            -------                                  -----------------
CommandArgument Test-AstObfuscation              ->      TE`s`t-AS`TOBFus`CAtIon
Member          ParameterSetName                 ->      ParamEterseTNAME
String          Set1                             ->      “S`et1”
Member          Position                         ->      PositiOn
Member          Mandatory                        ->      MAnDatoRY
Member          ValueFromPipelineByPropertyName  ->      ValUefroMPipELiNebyProPeRTyname
Variable        True                             ->      ${t`RuE}             
String          Parameter1                       ->      {“{0}{1}{2}” –f’Parame’,’te’,’r1’}
String          Param1                           ->      {“{1}{0}”-f’1’,’Param’}
String          ParamOne                         ->      {”{0}{2}{1}” –f ‘Para’,’e’,’mOn’}
Variable        ParameterOne                     ->      ${p`Ara`m`et`EROne}
Member          ParameterSetName                 ->      PaRamEtERsEtNaME
String          Set2                             ->      “SE`T2”
Member          Position                         ->      POSitiON
...
```

And gives a result that looks something like this:

![PSToken Obfuscation]({{site.baseurl}}/assets/images/example-token-obfuscation.png)

This form of PSToken obfuscation built into Invoke-Obfuscation is extremely good at breaking AMSI signatures (in fact I haven't seen a single instance that PSToken obfuscation does *not* break an AMSI signature). The real downside of PSToken obfuscation has to do with [something I mentioned in my last post]({{site.baseurel}}/PSAmsi-Minimizing-Obfuscation-To-Maximize-Stealth.html), obfuscation detection. PSToken based obfuscation adds lots of special characters and employs strange PowerShell syntax not often seen in the wild.

PSAmsi v1.0 aimed to remedy this by targeting PSToken based obfuscation only where it needed to be used, rather than using it on the entire script. This approach worked well, but can we do better?

### AbstractSyntaxTree Based Obfuscation

In PSAmsi v1.1, I've added a function called [Out-ObfuscatedAst](https://github.com/cobbr/PSAmsi/blob/master/PSAmsi/Obfuscators/PowerShell/Out-ObfuscatedAst.ps1) that utilizes the power of the AbstractSyntaxTree to perform stealthier obfuscation. The key to AST-based obfuscation is that it uses an AST's type **and it's location within the greater tree** as *context* for additional obfuscation options. A lot of these additional obfuscation options, at least so far, have to do with the order of ASTs within a script. This will make more sense with an example.

The entire tree is large and doesn't display nicely, so let's start by looking at a small tree that is just a portion of the larger tree that represents our example script:

![AST Obfuscation 0]({{site.baseurl}}/assets/images/tree-obfuscation-0.png)

This is an AttributeAst, just a line of code that applies various attributes to a parameter inside of a larger ParamBlockAst. The four children nodes of this Ast are all the attributes that are being applied to the parameter. One of these attributes is the "Mandatory" attribute, which indicates that a parameter is required to use the function. The Mandatory attribute is boolean, it can be either True or False. It turns out that we can specify True boolean attributes just by name (i.e. "Mandatory) or by actually assigning the True value (i.e. "Mandatory = $True"). Currently, we are specifying it only by name, so let's switch it:

![AST Obfuscation 1]({{site.baseurl}}/assets/images/tree-obfuscation-1.png)

We can do something very similar with the "ValueFromPipelineByPropertyName" attribute. This time we chop off the " = $True" part, and leave just the attribute name:

![AST Obfuscation 2]({{site.baseurl}}/assets/images/tree-obfuscation-2.png)

Finally, all of these children are just unrelated attributes being applied to a variable, so we can assign them in any order we want to, so we can rearrange them like this:

![AST Obfuscation 3]({{site.baseurl}}/assets/images/tree-obfuscation-3.png)

You can start to see how the context of an AST's location within the tree offers us additional obfuscation options. The knowledge that these Attributes are all children of a single AttributeAst applied to the same parameter allows us to reorder them inside the AttributeAst.

Now, let's zoom outwards a bit into the larger tree:

![AST Obfuscation 4]({{site.baseurl}}/assets/images/tree-obfuscation-4.png)

That is the original tree, which includes the previously discussed AttributeAst as a child. So let's first apply the obfuscation we've already done on that AttributeAst:

![AST Obfuscation 5]({{site.baseurl}}/assets/images/tree-obfuscation-5.png)

Now let's move on to look at the next AttributeAst that specifies several "Alias" names. This AttributeAst specifies alternate names, or "Aliases", that can be used instead of the default 'ParameterOne' parameter name. These aliases can of course be listed in any order, so we can switch those around:

![AST Obfuscation 6]({{site.baseurl}}/assets/images/tree-obfuscation-6.png)

We zoom outwards again into the larger tree:

![AST Obfuscation 7]({{site.baseurl}}/assets/images/tree-obfuscation-7.png)

We now see that there are two parameters in the larger ParamBlockAst. So far we have focused on the first ParameterOne parameter. So again we apply the obfuscation we have already found for this first parameter:

![AST Obfuscation 8]({{site.baseurl}}/assets/images/tree-obfuscation-8.png)

Of course, we can do very similar things to the second parameter, ParameterTwo. We can rearrange the Attributes in the AttributeAst of ParameterTwo, the same way we did with the first parameter:

![AST Obfuscation 9]({{site.baseurl}}/assets/images/tree-obfuscation-9.png)

But the parameters themselves can be reordered! So we switch the order of the entire ParameterAsts within the ParamBlockAst:

![AST Obfuscation 10]({{site.baseurl}}/assets/images/tree-obfuscation-10.png)

And now zoom out into the larger tree one last time:

![AST Obfuscation 11]({{site.baseurl}}/assets/images/tree-obfuscation-11.png)

This ScriptBlockAst contains the full ParamBlockAst we have been looking at so far, as well as three other NamedBockAst children. We first apply all of the obfuscation we have found for the ParamBlockAst:

![AST Obfuscation 12]({{site.baseurl}}/assets/images/tree-obfuscation-12.png)

Now we focus on the first NamedBlockAst child, the one containing the "Begin" block. All we are doing is assigning a variable called "Start" to a random value between minimum and maximum values. It turns out that there are other ways to assign a value to a variable than using the standard "=" operator. We can instead use the `Set-Variable` cmdlet. Additionally, named parameters (such as the `-Minimum` and `-Maximum` paramters to `Get-Random`) can be specified in any order we like. So we apply both of those techniques, using `Set-Variale` and switching the `-Minimum` and `-Maximum` parameters:

![AST Obfuscation 13]({{site.baseurl}}/assets/images/tree-obfuscation-13.png)

Moving on to the second NamedBlockAst child, the "Process" block, we can do something very similar using the `Set-Variable` to assign the result of an expression to the "Result" variable. Additionally, a numeric expression exists as a part of the expression being assigned to the "Result" variabble. It turns out that some numeric expressions, such as the addition operator, are "commutative" meaning they can also be reordered! So we apply both of those techniques:

![AST Obfuscation 14]({{site.baseurl}}/assets/images/tree-obfuscation-14.png)

Finally, the Begin, Process, and End NamedBlockAsts can be reordered within the function's ScriptBlockAst:

![AST Obfuscation 15]({{site.baseurl}}/assets/images/tree-obfuscation-15.png)

This gives us the final result from `Out-ObfuscatedAst`:

![Example AST Obfuscation]({{site.baseurl}}/assets/images/example-ast-obfuscation.png)

Hopefully, at this point you can see why I use "obfuscation" in quotation. This AST-based obfuscation doesn't really obscure what the script is doing much at all, it just finds functionally equivalent code that looks a little bit different, often by reordering children nodes within a tree. While it might not radically obscure what the code is doing, it changes it up *enough* that signatures will often break.

The cool thing about AST-based obfuscation is that it doesn't really use many special characters or strange syntax to make things looks different. By all accounts, it looks like normal PowerShell, which makes it very difficult to detect using any sort of obfuscation detection. I could have worked some of the special characters and strange syntax into AST obfuscation as well, but to me it makes more sense to keep that in PSToken obfuscation, so if you really want both you can just do both and get something crazy like this:

![AST and PSToken Obfuscation]({{site.baseurl}}/assets/images/example-both-obfuscation.png)

`Out-ObfuscatedAst` still has a lot of room for improvement. It's already at a place that it *can* break a lot of signatures, but the AST is a powerful structure and I am sure there are other good opportunites to introduce new forms of AST-based obfuscation into the function. I'll continue to try to add in new types as I find them.

### Within PSAmsi

Before, PSAmsi simply targeted PSToken-based obfuscation *only* at the signatures that needed to be obfuscated in an attempt to minimize the amount of obfuscation used and evade obfuscation detection techniques. Now, not only can we target obfuscation at only the signatures that need to be obfuscated, but we can use the stealthier AST form of obfuscation.

So, what PSAmsi does in the `Get-MinimallyObfuscated` function is:

1. First, enumerate all the signatures within the script.
2. Second, we iterate over each of the signatures. For each signature, we attempt AST-based obfuscation, which often succeeds at breaking the signature, but does not always.
3. Third, if AST-based obfuscation fails we can fallback to PSToken based obfuscation.

This approach allows us to limit the total amount of obfuscation we use, while still defeating all of the signatures present in a given script.