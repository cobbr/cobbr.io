---
layout: post
title: "Covenant: The Usability Update"
date: 2019-07-31 07:00:00 -0600
tags: Covenant GUI Web .NET dotnet C#
---

![Covenant Logo]({{site.baseurl}}/assets/images/covenant-logo.png){:height="200px" width="688px"}

## Intro

[Covenant v0.3](https:/github.com/cobbr/Covenant) is being released today and includes a brand new web-based interface. In this post, we will discuss Covenant's new interface and other important usability features.

Covenant has been under development for over a year, and the [initial public release](https://cobbr.io/Covenant.html) was over seven months ago. Development has been primarily focused on [new features and functionality](https://cobbr.io/Designing-Peer-To-Peer-C2.html). Until now, we've chosen to prioritize providing the features that an operator needs to succeed in modern red team engagements. But usability has not been a primary focus of the tool since its release. As it turns out, usability and documentation are actually *very* important features.

## Usability

If a feature is not intuitive or well-documented, the feature will likely not be used. Usability is particularly important for a collaborative command and control framework. It should be assumed that multiple operators are working together simultaneously and that not all operators are equally as familiar with the tool. If only the tool developer or team lead can decipher how to use the tool effectively, then the tool itself is not effective, no matter how many bells and whistles are in the feature list.

For Covenant, **usability will now be a primary focus**. This doesn't mean that every feature will immediately be intuitive, but that usability is now a primary focus of development. A feature will not be considered complete unless it is intuitive and well-documented.

## New Web-Based Interface

The first step towards a more usable and intuitive Covenant is a new web-based interface. Covenant has been almost totally transformed by the new interface, and is much more intuitive overall.

![Covenant Dashboard]({{site.baseurl}}/assets/images/covenant-gui-dashboard.png)

This post will showcase several of the most important upgrades in the new release and highlight some of the key features that have been enabled through the new web interface, primarily focusing on the usability benefits that the new features offer.

The interface is primarily driven through the sidebar, which includes the following navigation pages:

* **Dashboard** - The home view and the first thing you see when you login. You can get a quick glimpse of the active grunts you have obtained, the listeners that are currently active, and a few of the most recent taskings that have been assigned to grunts.
* **Listeners** - Provides an interface for managing listeners and listener profiles.
* **Launchers** - Provides an interface for creating, hosting, and downloading launchers to use in launching new grunts.
* **Grunts** - Displays a table for viewing all active and inactive grunts, as well as interacting with Grunts and assigning them new tasks. 
* **Tasks** - Provides an interface for editing and building new Tasks.
* **Taskings** - Displays a table for viewing all taskings that have been assigned to grunts.
* **Graph** - Provides a graph view for visualizing listeners, grunts, and the peer-to-peer graph.
* **Data** - Provides a view for data that has been collected from grunts during operation, such as credentials, indicators, and downloaded files.
* **Users** - Provides an interface for user management and creation.

All of these components have been improved in the new interface, but this post will focus on a few of the primary areas of improvement:

* User Registration and Management
* Listener and Profile Configuration
* Grunt Interaction
* Building Custom Tasks

## User Registration and Management

User registration has been streamlined in the new version and the web interface makes it much easier to create new users and modify existing users.

In previous versions, Covenant required an initial username and password to be passed on the command-line to kick things off. Now Covenant can be started without any parameters at all, and will allow you to register an initial user through the web interface. Once a user has been created, this option will no longer be available for security reasons.

![User Registration]({{site.baseurl}}/assets/images/covenant-gui-registration.png)

User management is also much improved by utilizing the new interface, and this update makes it much easier to set up users for collaboration. Registered users can be viewed and managed by navigating to the Users page on the sidebar:

![Users Table]({{site.baseurl}}/assets/images/covenant-gui-users.png)

[User registration and management](https://github.com/cobbr/Covenant/wiki/User-Management) is documented more thoroughly in Covenant's new [Wiki](https://github.com/cobbr/Covenant/wiki/Home).

## Listeners and Profiles

Listener creation and management is made much easier through the web interface and includes an entirely new profile building feature.

To create a new listener, navigate to the Listeners page on the sidebar, and click the "Create" button to start configuring the new listener.

![Configure Listener]({{site.baseurl}}/assets/images/covenant-gui-listenercreate.png)

Configure the new listener as desired, and click the "Create" button again, and your new listener will be started. We highly recommend configuring custom profiles for your listener or modifying existing profiles, because Covenant's default profiles could easily be signatured by defensive products.

In previous versions of Covenant, profiles needed to be developed prior to starting Covenant, but now profiles can easily be modified and created through the web interface.

To edit a pre-existing profile, navigate to the Listeners page on the sidebar, select the Profiles tab, and click on a pre-existing profile.

![Profile Table]({{site.baseurl}}/assets/images/covenant-gui-profiles.png)

The profile editing page allows you to change any aspect of the profile, including request and response headers, callback urls, and request and response formats. The user must be an Administrator to edit the HttpMessageTransform property for security purposes, and profiles applied to active listeners cannot be edited, though we would like to enable that feature sometime in the future.

![Edit Profile]({{site.baseurl}}/assets/images/covenant-gui-profileedit.png)

The ability to create and edit profiles through the interface is one of the more important new features, and should make it much easier to start customizing the way your network traffic communicates.

[Listener profile management](https://github.com/cobbr/Covenant/wiki/Listener-Profiles) and [listener management](https://github.com/cobbr/Covenant/wiki/Listeners) is documented more thoroughly in Covenant's new [Wiki](https://github.com/cobbr/Covenant/wiki/Home).

## Grunt Interaction

Grunt interaction is probably the most essential aspect of the Covenant project. Most of an operation will be conducted from this view and operators will spend most of their time interacting with grunts. For that reason, grunt interaction was a primary focus of the new interface, and needed to be as usable and intuitive as possible. The grunt interaction page includes realtime command updates via [SignalR](https://docs.microsoft.com/en-us/aspnet/signalr/overview/getting-started/introduction-to-signalr), tab-completable command suggestions via [Twitter typeahead](https://twitter.github.io/typeahead.js/), and multiple methods of interacting with grunts.

Navigating to the Grunts page in the sidebar will give you a list of active and inactive grunts.

![Grunt Table]({{site.baseurl}}/assets/images/covenant-gui-grunts.png)

Selecting an active grunt brings you to the grunt interaction page.

![Grunt Information]({{site.baseurl}}/assets/images/covenant-gui-gruntinfo.png)

The first thing you see on the interaction page is the Info tab, which will display all the info about the selected Grunt. There are several fields that are editable, such as Delay, JitterPercent, ConnectAttempts, and Note properties. Editing these values will change the value for the Grunt and will also cause a task to be applied to the Grunt to change this value on the implant itself.

Navigating to the Interact tab brings you to a command-line like interface. Operators will likely spend most of their time in this interface, which makes the usability of this view particularly important. The new command interface is an upgrade over previous iterations and much more intuitive. The interface is scrollable and vertically resizable, command suggestions are tab-completable as you type, and you will see updates in realtime so that you can see what other operators are doing.

![Grunt Command Interface]({{site.baseurl}}/assets/images/covenant-gui-gruntinteract.png)

Covenant has a new command-line parser that makes specifying complicated parameters on the command-line easier. However, if a command is too confusing to specify on the command-line, it can be useful to be more specific.

Navigating to the Task tab reveals a dropdown dialog where you can select a specific task that you would like to execute and allows you to fill parameters into the dialog boxes to ensure that parsing will not get in your way.

![Task Tab]({{site.baseurl}}/assets/images/covenant-gui-grunttask.png)

The Taskings tab shows the grunt's command history in a searchable table view, which allows you to click on labels to get more detailed information for any specific tasking, task, or user.

![Grunt Tasking Table]({{site.baseurl}}/assets/images/covenant-gui-grunttaskings.png)

[Grunt interaction](https://github.com/cobbr/Covenant/wiki/Grunt-Interaction) is documented more thoroughly in Covenant's new [Wiki](https://github.com/cobbr/Covenant/wiki/Home).

## Building Custom Tasks

Building custom tasks has been another core focus for the new interface. In previous versions of Covenant, implementing custom tasks was not a simple process, but now operators can add new tasks or edit existing tasks at any time during the operation and begin using them immediately.

Covenant provides a rich object model for building new tasks. Navigating to the Tasks page on the sidebar reveals tables that show all Tasks, ReferenceSourceLibraries, ReferenceAssemblies, and EmbeddedResources.

A user can use any of these existing components to help building new tasks. For instance when creating a new task, a user could add the "SharpSploit" ReferenceSourceLibrary and begin using any SharpSploit namespace to help complete the given task.

![Task Table]({{site.baseurl}}/assets/images/covenant-gui-tasks.png)

To create a new Task, click the Create button at the bottom of the Tasks tab.

![Create Task]({{site.baseurl}}/assets/images/covenant-gui-taskcreate.png)

Then you can configure the Task as needed, click the Create button, and the Task should now be usable within any of your grunts.

![Use Task]({{site.baseurl}}/assets/images/covenant-gui-taskuse.png)

[Building tasks](https://github.com/cobbr/Covenant/wiki/Building-Tasks) is documented more thoroughly in Covenant's new [Wiki](https://github.com/cobbr/Covenant/wiki/Home).

## Goodbye for now, Elite

The new interface also means that we have to say goodbye, at least for now, to Covenant's client-side interface, [Elite](https://github.com/cobbr/Elite). It's never fun to throw away large chunks of hard-earned, functioning code, but in making a move towards usability, we determined that a command-line interface was too limiting for our current purposes.

However, I do think there is value in having a command-line client and Elite may make a return at some point in the future. But for now, we've chosen to focus our development efforts on Covenant's new interface.

Elite will continue to reside at [https://github.com/cobbr/Elite](https://github.com/cobbr/Elite) for historical purposes, in case someone wants to use an older version of Covenant for some reason. Eventually the repository may be brought up to date in later versions of Covenant.

![Elite death?]({{site.baseurl}}/assets/images/covenant-elite-death.png)

## Documentation

In addition to the new interface, Covenant was in need of thorough documentation.

Covenant has a new, revamped [Wiki](https://github.com/cobbr/Covenant/wiki) that documents Covenant's features, many of which were not detailed in this post. The Wiki currently documents the majority of Covenant's core features and will continue to be updated as the project evolves. Additionally, we will be recording a video walkthrough series that demonstrates Covenant's features in the near future.

## Future Developments

We are excited to be releasing Covenant's new interface and are happy with its current state, but we are always thinking about future additions and improvements we could make. In the near-term future, we will be continuing to polish Covenant's documentation, interface, and usability features. We are hoping that user feedback will help us fill in the gaps and really get the interface right before moving on to new features.

We do have a few ideas about future UI improvement, such as improving the Graph view, possibly introducing Microsoft's new [Blazor](https://docs.microsoft.com/en-us/aspnet/core/blazor/index?view=aspnetcore-3.0) single-page application framework, and continuing to enhance the implementation of Microsoft's [SignalR](https://docs.microsoft.com/en-us/aspnet/signalr/overview/getting-started/introduction-to-signalr) framework for real-time updates.

But we are hoping that most of the future UI improvements come from user feedback! If you are a regular Covenant user or haven't tried out Covenant in a while, we hope you will try out the new interface and let us know what you think. As always, feel free to join the discussion in the #covenant channel in the [BloodHound Gang Slack](https://bloodhoundgang.herokuapp.com/)!