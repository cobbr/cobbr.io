---
layout: post
title: "SharpShell: The Worst Scripting Engine of All-Time"
date: 2018-12-11 07:00:00 -0600
tags: SharpSploit SharpShell .NET dotnet C#
---

In [my last post]({{site.baseurl}}/SharpGen.html) I introduced [SharpGen](https://github.com/cobbr/SharpGen), a .NET Core console application that utilizes the Roslyn C# compiler to quickly cross-compile .NET Framework console applications and libraries. In this post I'll introduce a very similar, but slightly different tool: [SharpShell](https://github.com/cobbr/SharpShell)!

To preface, I view [SharpShell](https://github.com/cobbr/SharpShell) mostly as a fun proof-of-concept that demonstrates some C# tradecraft possibilities for future operational projects. I view [SharpGen](https://github.com/cobbr/SharpGen) as a more operationally useful tool.

# SharpShell

The idea of [SharpShell](https://github.com/cobbr/SharpShell) is very similar to SharpGen, in that it uses the [Roslyn](https://github.com/dotnet/roslyn) C# compiler to cross-compile .NET Framework assemblies. However, unlike SharpGen, SharpShell is designed to compile *and* execute the compiled assemblies in memory. SharpShell provides a rudimentary shell-like interface and acts as a **very** basic scripting engine.

By very basic, I mean very basic. Every one-liner exists on it's own, no variables are carried over between one-liners, and it has no tab-completion or any of the other nice features or syntax you might be used to having in a legitimate scripting engine. `SharpShell` essentially just takes C# one liners, embeds them into a class, compiles them into assemblies, executes them, and outputs the results as text. This is why I like to refer to it as the worst scripting engine of all time.

SharpShell is broken up into three distinct C# projects:

* **SharpShell** - `SharpShell` is the most straightforward of the three projects. It acts as an interactive shell and scripting engine for C# code compiled against chosen source code, references, and resources. The main caveat with `SharpShell` is that it depends upon .NET Framework 4.6 and 3.5/4.0 being installed on the system. This is because `SharpShell` depends upon the Roslyn API, which requires 4.6, and executes an assembly in memory cross-compiled for you choice of versions 3.5 or 4.0.
* **SharpShell.API** - `SharpShell.API` and `SharpShell.API.SharpShell` are two projects meant to be used in tandem. To avoid the opsec limitations and .NET Framework 4.6 requirements of `SharpShell`, `SharpShell.API` acts as a web-server that handles the compilation for `SharpShell.API.SharpShell`. `SharpShell.API` is a ASP.NET Core 2.1 application and is cross-platform.
* **SharpShell.API.SharpShell** - `SharpShell.API.SharpShell` provides the same interface as `SharpShell`, but doesn't have the .NET Framework 4.6 requirement. `SharpShell.API.SharpShell` runs on .NET Framework 3.5, but also requires network communication with a `SharpShell.API` server for handling compilation of assemblies.

## Standalone SharpShell

Let's walkthrough an example using the standalone version of `SharpShell`. We start up `SharpShell` and are given a shell-like interface:

![SharpShell]({{site.baseurl}}/assets/images/sharpshell.png)

As the name suggests, we can execute C# code through this shell interface:

![SharpShell Example 1]({{site.baseurl}}/assets/images/sharpshell-example-1.png)

`SharpShell` expects a `return` statement somewhere in your code, and will transform the returned value into a string and display it as output:

![SharpShell Example 2]({{site.baseurl}}/assets/images/sharpshell-example-2.png)

You can specify multi-line code strings by using the `\` escape symbol. The logic can be as complex as you like, as long as you have a `return` statement:

![SharpShell Example 3]({{site.baseurl}}/assets/images/sharpshell-example-3.png)

`SharpShell` starts to become more powerful once combining this shell-like interface with the capabilities of SharpGen, such as compiling against chosen source code, references, and resources. `SharpShell` by default compiles against the `SharpSploit` source code:

![SharpShell Example 4]({{site.baseurl}}/assets/images/sharpshell-example-4.png)

For simple one-liners, `SharpShell` will add the return statement for you:

![SharpShell Example 5]({{site.baseurl}}/assets/images/sharpshell-example-5.png)

`SharpShell` can be configured with custom source code, references, and embedded resources in just the same way as SharpGen by editting YAML configuration files. Reference the [SharpGen blog post for configuration details]({{site.baseurl}}/SharpGen.html).

Finally, `SharpShell` will print out the compilation errors in the event that you experience a typo (I'm sure this will never happen):

![SharpShell Example 6]({{site.baseurl}}/assets/images/sharpshell-example-6.png)

### Opsec Considerations

There are some important opsec considerations when using the standalone `SharpShell`. First, there is the obvious consideration that the `SharpShell` binary and all configuration files have to be present on disk. As mentioned before, `SharpShell` is a proof-of-concept. However, `SharpShell` could fairly easily be weaponized to remove all need for those files to be present on disk.

When developing `SharpShell`, I also assumed that conducting the compilation on the target system itself would result in temporary compilation files being written to disk. This is what happens when compiling with `csc.exe`, PowerShell's `Add-Type`, or the `System.CodeDom.Compiler` namespace. For instance, an example of using PowerShell's `Add-Type` module:

![ProcMon Add-Type]({{site.baseurl}}/assets/images/procmon-addtype.png)

With ProcMon, we can see that `powershell.exe` spawns off `csc.exe` and writes several temporary files in the `%TEMP%` directory, including a `*.cs` file containing the source code to be compiled. These files are quickly deleted, but appear in a predictable location that could allow an EDR or other endpoint product to create signatures on the compilation of malicious code.

During development, I was under the impression that `SharpShell` would suffer from this opsec limitation as well. If that were the case, this would be an opsec limitation that would be difficult to avoid. However, I discovered that by using the [Roslyn API](https://github.com/dotnet/roslyn), we avoid this limitation. This library does not spawn csc.exe and compiles truly in memory.

## SharpShell.API

`SharpShell.API` is an extremely simple API that abstracts the compilation process away from the client-side project, `SharpShell.API.SharpShell`. `SharpShell.API` is a ASP.NET Core 2.1 application, so you must install dotnet core to use this project. However, this also makes the project cross-platform.

You can start `SharpShell.API` by using the `dotnet` command-line tool: `dotnet .\SharpSploit.API.dll`

This starts up a locally-bound web server on port 5000. `SharpSploit.API` also includes a `Swagger` page for interactive API usage:

![Swagger]({{site.baseurl}}/assets/images/swagger-sharpshell.png)

`SharpShell.API` is very simple and only includes a single API endpoint that accepts compilation requests and returns the compiled assembly. All the configuration details are included in the compilation requests:

![Compilation API]({{site.baseurl}}/assets/images/sharpshell-compilation-api.png)

`SharpShell.API.SharpShell` uses this web server to request source code to be compiled. `SharpShell.API` is really only meant to be used for this simple proof-of-concept. `SharpShell.API` is not C2, please don't use `SharpShell.API` as C2.

## SharpShell.API.SharpShell

I created `SharpShell.API.SharpShell` as a proof-of-concept that does not have a dependency on .NET Framework 4.6, runs only on .NET Framework 3.5, doesn't write or expect any extraneous files on disk, and still provides the same shell interface as the standalone `SharpShell`.

Once `SharpShell.API` has been started, we can start `SharpShell.API.SharpShell` and it will connect to our `SharpShell.API` instance and function just the same as `SharpShell`:

![SharpShell.API.SharpShell Example 1]({{site.baseurl}}/assets/images/sharpshell-api-sharpshell-example-1.png)

We can also point `SharpShell.API.SharpShell` at a remote address to communicate with a remote `SharpShell.API`:

![SharpShell.API.SharpShell Example 2]({{site.baseurl}}/assets/images/sharpshell-api-sharpshell-example-2.png)

# Detection

While Microsoft doesn't appear to yet have any guidance on .NET optics or detection, it does appear that they are beginning to implement these optics. [Matt Graeber](https://twitter.com/mattifestation) discovered an undocumented ETW provider that could be utilized to begin taking advantage of these optics:

![.NET Optics]({{site.baseurl}}/assets/images/net-optics.png)

Matt published a script for collecting these type of events [here](https://gist.github.com/mattifestation/444323cb669e4747373833c5529b29fb).

In order to demonstrate how this ETW provider could be utilized to detect `SharpShell` activity, I published a slightly modified copy of Matt's script to make it a bit easier to use and is available [here](https://gist.github.com/cobbr/1bab9e175ebbc6ff93cc5875c69ecc50). I'll be using this script to in the examples below.

First, we'll start the collector by using the `Start-DotNetEventCollection` method:

![.NET Collector]({{site.baseurl}}/assets/images/start-collector.png)

Now that we have begun event collection, we will use `SharpShell` to execute a shell command and use the Mimikatz coffee command:

![SharpShell Collection]({{site.baseurl}}/assets/images/sharpshell-collection.png)

We will use the `Get-DotNetEvents` command to gather .NET events from the specified process:

![Event Collection]({{site.baseurl}}/assets/images/event-collection.png)

You can see in the `AssembliesLoaded` property the `SharpShell` and `Microsoft.CodeAnalysis` (Roslyn) assemblies being loaded. While this would not be a particularly resilient detection, defenders could look for these assemblies being loaded to detect `SharpShell` or assembly compilation activity.

You'll also be able to look through any of the other properties collected by Matt's script:

![Event Collection 2]({{site.baseurl}}/assets/images/event-collection-2.png)

While this script may not be scalable to the enterprise level, defenders could look to collect this type of data using something like [KrabsETW](https://github.com/Microsoft/krabsetw).

For more information on .NET optics and detection, I'd recommend watching [Luke Jenning's recent talk at Bluehat that covers this topic](https://www.youtube.com/watch?v=02fL2xpR7IM).

# Summary

To describe it succinctly, `SharpShell` is best for convenience and ease-of-use as a simple proof-of-concept C# scripting engine, and `SharpShell.API` and `SharpShell.API.SharpShell` are a proof-of-concept of how this could be properly weaponized.

I see `SharpShell.API` and `SharpShell.API.SharpShell` as an example of what could be accomplished over a legitimate C2 channel. Even though `SharpShell` does accomplish compilation on the target without writing temporary files to disk, I still think it is always safer to compile on the server-side, since there is no real benefit from compiling on the target. And in realistic scenarios, we typically have a convenient C2 communication channel we can use to bring down pre-compiled assemblies to be executed.

# References

* Roslyn - [https://github.com/dotnet/roslyn](https://github.com/dotnet/roslyn)
* SharpSploit - [https://sharpsploit.cobbr.io/api](https://sharpsploit.cobbr.io/api)
* SharpGen - [https://cobbr.io/SharpGen.html]({{site.baseurl}}/SharpGen.html) and [https://github.com/cobbr/SharpGen](https://github.com/cobbr/SharpGen)
* CollectDotNetEvents.ps1 (original) - [https://gist.github.com/mattifestation/444323cb669e4747373833c5529b29fb](https://gist.github.com/mattifestation/444323cb669e4747373833c5529b29fb)
* CollectDotNetEvents.ps1 (modified) - [https://gist.github.com/cobbr/1bab9e175ebbc6ff93cc5875c69ecc50](https://gist.github.com/cobbr/1bab9e175ebbc6ff93cc5875c69ecc50)
* KrabsETW - [https://github.com/Microsoft/krabsetw](https://github.com/Microsoft/krabsetw)
* Luke Jennings, Bluehat v18 (covers .NET optics) - [https://www.youtube.com/watch?v=02fL2xpR7IM](https://www.youtube.com/watch?v=02fL2xpR7IM)