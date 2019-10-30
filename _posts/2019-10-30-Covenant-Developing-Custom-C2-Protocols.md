---
layout: post
title: "Covenant: Developing Custom C2 Communication Protocols"
date: 2019-10-30 07:00:00 -0600
tags: Covenant C2Bridge TCP HTTP C2 .NET dotnet C#
---

As of [Covenant v0.4](https://github.com/cobbr/Covenant), Covenant provides options that allow developers to integrate custom C2 communication protocols into an operation within Covenant.

This post provides a guide for Listener development, introduces the new [C2Bridge](https://github.com/cobbr/C2Bridge) project, and describes how C2Bridges can be used within Covenant.

### HTTP Protocol

For C2 (Command and Control), we require some communication protocol with which implants can interact with an attacker-controlled server to receive taskings and report back the results of issued commands. There is nearly an infinite amount of protocols we could use to accomplish this communication, but not all of them are convenient or common enough to blend into target environments.

Covenant officially supports only a single communication protocol: **HTTP**.

There are many benefits to using HTTP for this implant to server communication. HTTP is a ubiquitous protocol that is likely to blend into many target environments. [Domain Fronting](https://blog.cobaltstrike.com/2017/02/06/high-reputation-redirectors-and-domain-fronting/) can be utilized via HTTP, allowing us to utilize high trust domain names. HTTP is high-throughput and low latency, allowing us to quickly transfer large amounts of data between the server and an implant. For these reasons, many C2 frameworks make use of the HTTP protocol for its primary communication protocol.

But there are also drawbacks to using the HTTP protocol. For example, Covenant's HTTP listener requires a Grunt implant to continuously poll the server to check for new taskings, even while there are none.

![Empty HTTP Polling]({{site.baseurl}}/assets/images/diagram-http-poll.png)

This consistent, unnecessary traffic is not always desirable. Many C2 frameworks have a similar drawback with their own HTTP implementation.

### Protocol Agnostic

There really is no reason that Covenant should need to constrain us to this one communication protocol. As long as you are able to transfer data to and from an implant on a remote network, Covenant should ideally be agnostic to the communication protocol that accomplishes this data transfer.

![Protocol Agnostic]({{site.baseurl}}/assets/images/diagram-unknown-protocol.png)

So while Covenant only includes the one built-in protocol for now, it provides developers with the tools necessary to integrate custom communication protocols into the platform.

Covenant now supports two primary features for integrating new communication protocols: 

1. **Listeners** - Listeners can be created from within the Covenant interface and are built-in to Covenant directly. You may already be familiar with Listeners if you have used the Covenant platform before.
2. **C2Bridges** - C2Bridges are a new feature that are developed outside of the Covenant platform entirely.

## Creating New Listeners

If a developer would like to create a new first-class listener that integrates with the Covenant interface or if they would like to contribute a communication protocol to the Covenant project as a built-in listener, they should choose to create a Listener rather than a C2Bridge.

Creating a Listener requires that the developer write their communication protocol logic in C# within the Covenant project, and that they write a class that inherits from the `Listener` class within Covenant. The developer will have to make some minor edits to the Covenant backend code to accomplish this. The developer will also need to make some changes to Covenant's front-end code so that users can visually interact with and create new Listeners that use the new communication protocol.

When developing a new communication protocol, a developer needs to handle:

1. **Listener-side development**
2. **Implant-side development**

This is true for both Listeners and C2Bridges. Each side of the protocol needs to know how to "speak" the new protocol.

### Listener-Side Development

As of Covenant v0.4, the development of new listeners is much easier than before. Listeners only need to implement a mechanism to read data from implants and write data back to implants. All of the details for interpreting the data returned from implants is handled by Covenant itself, listeners do not need to handle this.

For example, the existing relevant HTTP listener code is only a handful of lines of code (though it does make use of ASP.NET Core to offload development):

{% gist 3af142374722a0384bf1c09642d34876 %}

Let's take a quick look at the `Post()` function in particular. This is the function that gets called each time an implant writes data to the HTTP listener. Whenever an implant writes data, the listener is responsible for: 

1. Determining which implant this data is from. The Listener determines which implant is posting data by using the `GetGuid()` helper function, which parses the HTTP request and locates the Grunt GUID value within the data by using the configured location within the `HttpProfile`.
2. Retrieving the actual data from within the larger message. Profiles determine where the data is placed within an HTTP message. This allows us to create benign looking HTTP messages that our data is hidden within. The Listener is responsible for extracting the data from within the message, using the `Profile` to parse out this information. This is done within the `Post()` method with the `body.ParseExact()` method.
3. Writing the extracted data to an `InternalListener` object. The `InternalListener` object will interpret messages from implants, so that the listener doesn't have to. The `Post()` methods calls the `InternalListener.Write()` method to forward the extracted message to Covenant, which will handle everything that needs to happen when an implant returns data.
4. The Listener needs to craft a response to send to the implant. If there are any available tasks for the implant, this is where the Listener can inform the implant of these tasks. Listeners do not need to track available tasks, the `InternalListener` object again will handle this for us. Calling the `InternalListener.Read()` method will retrieve any data that is available for the implant.
5. Finally, the Listener needs to format this data into the format that the Profile requires. The HTTP Listener accomplishes this with the `String.Format()` method.

**Front End Development**

In addition to the logic that powers the Listener, Covenant's front-end should be modified so that the new Listener is easy to create and interact with.

Developers should follow the examples of the `HttpListener` and `BridgeListener` classes to get an understanding of the front-end code that needs to be added. Specifically, developers should add the listener to the list in the [Create.cshtml view](https://github.com/cobbr/Covenant/blob/master/Covenant/Views/Listener/Create.cshtml).

Once this is done correctly, your new Listener should appear in a new tab next to the `HttpListener` and `BridgeListener`:

![New Listener Tab]({{site.baseurl}}/assets/images/covenant-gui-listenercreatenew.png)

### Implant-Side Development

Secondly, a developer must write a new implant that utilizes the new communication protocol to receive new taskings. Developers can make use of the majority of the code from existing implants, but must replace the relevant code for an implant to utilize the new protocol.

To do so, a developer can create a new `ImplantTemplate`. It is relatively easy to create a new `ImplantTemplate` through the Covenant interface. Navigating to the "Templates" navigation page will display a table of the existing implants:

![Templates Navigation Page]({{site.baseurl}}/assets/images/covenant-gui-templates.png)

The built-in templates are GruntHTTP and GruntSMB, which utilize the HTTP and SMB communication protocols respectively. When it comes to listeners, we are only concerned with *outbound* protocols (i.e. protocols that traverse the external edge of the network), not peer-to-peer protocols such as SMB.

To create a new `ImplantTemplate`, we can click the "Create" button underneath the table.

![Create New Template]({{site.baseurl}}/assets/images/covenant-gui-templatecreate.png)

Here, we can place the C# code of our new implant that utilizes our new communication protocol. We must specify the code for the stager as well as the full implant. Much of this code can be copied from existing implants, and we can simply replace the portion of the code that handles communication with the listener.

Once, we've created the new `ImplantTemplate`, we can see that it appears within the table:

![New Template Created]({{site.baseurl}}/assets/images/covenant-gui-templatenew.png)

The benefit of creating a Listener is that you can provide users with an intuitive interface to provide options needed for the Listener to function and everything is tied into Covenant's natural workflow. The drawback is the extra development work needed to edit Covenant's backend and frontend codebase.

## Creating C2Bridges

C2Bridges are a new feature introduced in Covenant v0.4. The C2Bridge feature is meant to allow a developer to quickly create new communication protocols without having to edit any Covenant code at all, and quickly make use of their creation within Covenant.

![C2Bridge Architecture]({{site.baseurl}}/assets/images/diagram-c2bridge.png)

Again, just as with Listeners, a developer needs to handle both the implant-side of development as well as the listener-side.

### Listener-Side Development

Covenant contains a built-in "Listener" called a `BridgeListener` that facilitates plugging a new protocol into the Covenant platform. A `BridgeListener` is a simple TCP listener that should be used as a means to transfer data between the new C2Bridge and Covenant.

Developers can use the new [C2Bridge](https://github.com/cobbr/C2Bridge) project as a template for creating new C2Bridges. This is not a requirement, but the C2Bridge project provides helpful functionality to connect your new protocol to a `BridgeListener` without any additional work.

Within the C2Bridge project, there is an abstract `C2Bridge` class. A developer can inherit from this class and implement the needed functions to read and write from implants to the `BridgeListener` using the new chosen communication protocol.

{% gist bcf8711d0dc6396b0e4905b55c5f797a %}

This abstract `C2Bridge` class is well-documented to make it as easy as possible for developers. The entire purpose of the C2Bridges project is to allow developers to focus on developing their new communication protocol, while focusing on Covenant integration as little as possible.

The C2Bridges project also provides an example `TcpC2Bridge` class that inherits from the `C2Bridge` class that demonstrates how to develop a C2Bridge. The `TcpC2Bridge` provides a new outbound communication protocol that utilizes TCP for C2. An outbound TCP C2 protocol probably has limited usefulness in a real offensive operation, but this is available primarily as an example of how to create a C2Bridge.

Examining the `TcpC2Bridge` shows how to construct a C2Bridge:

{% gist 969467644f7c746838d3b003cb0908d9 %}

In the `RunAsync()` function, we start our `TcpListener` and continually wait for new clients. For every client, we continually read data from the implant and write that data to Covenant using the `WriteToConnector()` helper function that handles this for us.

The `OnReadBridge()` function handles when Covenant is writing data back down to the implant. Whenever, we read data from the "bridge" (Covenant), this function gets called. When this happens, we determine which implant this data is for and write it down to the corresponding implant `TcpClient`.

To use this `TcpC2Bridge` within Covenant, we first need to create a `BridgeListener` within Covenant. To do so, we first navigate to the Listeners navigation page:

![Listeners Navigation Page]({{site.baseurl}}/assets/images/covenant-gui-listeners2.png)

We then click on the "Create" button and click on the BridgeListener tab at the top of the screen:

![Create a BridgeListener]({{site.baseurl}}/assets/images/covenant-gui-listenercreatebridge.png)

The following options will need to be configured when creating the `BridgeListener`:

* **Name** - The `Name` of the listener that will be used throughout the interface. Pick something recognizable!
* **BindAddress** - The `BindAddress` is the local ip address that the listener will bind to. This can be helpful in cases where the Covenant host has multiple NICs. Usually, this value will be `0.0.0.0`.
* **BindPort** - The `BindPort` is the local port that the listener will bind to. This is the port that the C2Bridge will connect to.
* **ConnectPort** - The `ConnectPort` is the *callback* port that Grunts will be directly connecting to. This is the port the C2Bridge should listen on. This may or may not be relevant depending upon the protocol used by the C2Bridge.
* **ConnectAddress** - The `ConnectAddress` is the *callback* address that Grunts will be directly connecting to. This should be the external address of the C2Bridge, or if you are using redirectors this should be the address that points to the external redirector.
* **BridgeProfile** - The `BridgeProfile` determines the behavior of Grunt and Listener communication. **The profile must specify BridgeMessengerCode that is compatible with the C2Bridge you choose to use.**

You'll notice that a `BridgeProfile` is required to create a `BridgeListener`. A `BridgeProfile` is similar to the `HttpProfile` you may be familiar with already. It allows us to configure what our network traffic looks like on the wire, and specify where the data should be stored within the protocol.

### Implant-Side Development

A `BridgeProfile` has one unique feature not found in the `HttpProfile`, it requires a `BridgeMessengerCode` property. This property allows us to specify the *implant code* used to communicate with our new C2Bridge. When we created the C2Bridge, we defined a "listener" that implants can communicate with for C2, but the implant must also know how to communicate over this new protocol.

Some C2 frameworks attempt to solve this problem with an intermediate agent that sits between the implant and the listener that translates messages to the new protocol.

![Integrating New Protocols - Translation Agent Approach]({{site.baseurl}}/assets/images/diagram-translation-agent.png)

However, this solutions has its drawbacks. This requires us to run a secondary agent for each implant, and we aren't able to utilize our new protocol between the implant and the translation agent.

Covenant solves this problem slightly differently. In Covenant, the new communication code gets embedded within the implant itself. This requires less overhead than the translation agent approach, and does not require an additional agent to be executed within the target network.

![Integrating New Protocols - C2Bridge Approach]({{site.baseurl}}/assets/images/diagram-covenant-bridgemessengercode.png)

This new communication code is what is specified in the `BridgeMessengerCode` property of the `BridgeProfile`. For example, the `BridgeMessengerCode` for the `TcpBridgeProfile` is as follows:

{% gist b71d5858278653392d8baa1cf6483fa8 %}

A `BridgeProfile` must include the unmodified `IMessenger` interface, as well as a `BridgeMessenger` class that inherits from this interface. The `BridgeMessenger` should implement all of the code to read and write data to and from the C2Bridge.

Once again, the TCP example here is meant to serve as a guide to developers wanting to develop their own protocol, not necessarily as something to use within an operation.

For more information, please reference the Covenant wiki pages for [C2Bridges](https://github.com/cobbr/Covenant/wiki/C2Bridges) and [BridgeListeners](https://github.com/cobbr/Covenant/wiki/Bridge-Listeners).

## Conclusion

As an operator, it's easy to fall into a trap of always defaulting to a standard polling HTTP protocol. There's nothing wrong with using an HTTP communication protocol, it can be highly effective, but there are many other options out there to consider.

Mixing up the protocols you use as an operator could be eye-opening and beneficial to your blue team. While Covenant does not provide many communication protocols out of the box, it provides the tools that enable the integration of any communication protocol you could want to use.

For example, you could implement a DNS C2Bridge, C2Bridges that integrate with 3rd party websites such as Twitter, Github, Slack, and Dropbox, or get more creative with Amazon SQS, Amazon S3 buckets, or Microsoft Outlook. The only limit is your imagination and a bit of development time!

### Additional Resources

* **Flying a False Flag: Advanced C2, Trust Conflicts, and Domain Takeover - Nick Landers** - I would highly recommend [this 2019 Blackhat talk](https://www.youtube.com/watch?v=2BEwqbCbQuM) to anyone looking for some creative ideas for new communication protocols.
* **C3 (Custom Command and Control)** - [C3](https://github.com/FSecureLABS/C3) is a platform-agnostic toolset that attempts to solve a similar problem.