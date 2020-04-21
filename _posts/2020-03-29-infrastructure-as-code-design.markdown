---
layout: post
title: Infrastructure as Code Design
date: '2020-03-29 22:58:17'
categories: IaC
---

SOLID principles are the cornerstone of object-oriented programming and for good reason. These principles allow us to produce flexible, testable, and clearly defined digital nuggets that can continuously provide customer value throughout their lifetime.

Why would this be interesting to Infrastructure as Code peeps? Software development, in my simple-minded view, has already gone through the shenanigans of shitty, untested, non-extensible code. The phone calls to developers at midnight are all but gone. But while these struggles and growing pains occurred in the world of devs, the ops folks were SSHing and RDPing away to install files and install packages, chatting away about filesystems, kernel parameters, and networking, while the server parks grew and digitalization journeys began to grow exponentially larger than the amount of technicians that could SSH in and turn those digitalization journeys into reality. Puppet, Chef, Saltstack, and Ansibile began to resolve a lot of the problem in ops by providing a toolset to attack their infrastructure as if it was code and thus begin the journey for many an Ops techy down the road of code development. Unfortunately, that journey is fraught with the same pitfalls of shitty, untested, non-extensible code. Did we really think we wouldn't have the same growing pains?

In the following sections, I'll take a look at how to apply SOLID concepts to IaC, the natural evolution of an IaC monolith, the pitfalls of that evolution, and some ways to address those pitfalls.

## SOLID

SOLID is an mneumonic acronym for:

- Single-responsibility principle
- Open-closed principle
- Liskov substituion principle
- Interface segmentation principle
- Dependency inversion principle

### Single-responsibility principle

A class should do one thing and only one thing.

### Open-closed principle

A class should be open for extension, but closed for modification.

### Liskov substitution principle

An object should be able to be replaced by any of its subtypes, without altering the subtypes behavior. We'll focus mostly on design by contract, which is the basis for this principle.

### Interface segmentation principle

Client specific interfaces are better than broad catch-all interfaces.

### Dependency inversion principle

Modules should depend on abstractions. Programs should not be built like pyramids, but rather each building block should have a clear abstraction layer.

## Infrastructure as Code

SOLID principles are used as the basis for virtually all, if not all, object oriented software. These principles exist so that software can be more flexible, maintainable, and usable. This guide will attempt to apply these same methods to Infrastructure as Code.

### Problem - Day 1

If you're one of those lucky people that have gone cloud-only, your Day 1 has probably gone the way of the telegram. For those of us stuck in the world of virtual machines, SSH and RDP, you're stuck with operating a lot of shit. This shit is the on-site Sharepoint and Exchange servers, the on-site Elasticsearch and Splunks clusters, the on-site Oracle RAC clusters, the on-site IIS servers and butt loads of random MSSQL servers. All of those servers need to get provisioned and that expensive software needs to get installed. In my experience at one of the largest healthcare providers in Scandinavia, the pain point simply becomes not having enough competent hands. The digitalization journey is just running at a faster pace than those hands can type. Something has to give.

This is where configuration management tools come in and save the day. Ansible, Puppet, Chef, or Saltstack provide toolsets so that these hands can begin to program up their infrastructure. Upgrades in the future can start with an existing code base. Those lovely Windows end-of-life projects can now simply begin with the click of a button. Those new public SSH keys can be installed on all of those hundreds of Unix servers with a click. Everybody breathes a sigh of relief and some of that pain is gone. We never thought we'd make it and yet now we just automated some of this shit that was taking forever to do manually.

### Problem - Day 2

Things are going along smoothly. Your server though turn yellow in Nagios, warning that 30 certificates are going to expire in 14 days, so you send away a CSR to Verisign and your feeling cool. You got this shit automated. You'll just... wait. Shit. We didn't encrypt anything.

What do you do when you need to modify your PowerShell script that it is well over 2 000 lines of code? What do you do when you realize that thing you're automatically can't be tested? And that thing has now become VERY important for the life of the enterprise? What do you do when you realize that only you can use your code? These are not problems I've heard on the streets. These are problems I've experienced first hand. I've also come to understand that these problems are not unique to IaC. In fact, SOLID principles were designed 30 years ago to tackle these specific software design problems. Here, we're going to use these principles to tack IaC design problems.

### IaC design

You should be using some way to abstract your IaC modules. You should never be building code to get the status of a service or install a package. These code nuggets should be abstracted, tested, and have a clear interface and then, and only then, can these nuggets be used to build something greater. 

In Puppet this is applied with "roles and profiles". In Ansibile, it's applied with "roles". Although I don't know every IaC tool out there, I would argue that if you do not have the ability to create a tower will Legos, but are rather forced to mold your own tower, you would probably be best served looking elsewhere.

"Role and profiles" in Puppet and "roles" in Ansible allow you to build modules with clearly defined interfaces. For example, you can create a MySQL module. You can create an Oracle module. You can create an IIS module. You can then tie these modules together to produce some sort of business value.

This type of design attempts to fullfil these principles. Each module has a single responsibility. Each module is open for extension (we can for example add a provider to for Gentoo or FreeBSD), but it's closed for modification. Each module has a specific contract and a specific client interface. Finally, are dependency is only on the interface of these modules, not on their concretions.

### IaC design gone to shit

We can use SOLID design principles to create well-defined, manageable modules, but what happens when your monolith get unruly? A few applications become a hundred. Even a few people merging to the same repsoitory can quickly escalate into merge hell.

#### Release your infrastructure

Your "roles" module or your "profiles" module needs to be released like regular code. Tag it. Create a CHANGELOG. Create a release candidate and promote it eventually to production.

#### Choose a branching strategy

Choose GitFlow, GitHubFlow, GitLabFlow... whatever. Choose a flow and stick with it. All of these strategies requires branching. Branch your feature and submit merge requests. Your team will be happy and those pitfalls should start to magically disappear.

#### Unit test your modules

If you are going to have a well-defined contract that is going to do one specific thing, it's import to test the various ways someone might be able to call that interface, so that you can be assured that your module delivers what it says it will deliver. What happens if you try to install a package on Gentoo, SUSE, RedHat, Ubuntu... What happens if I try to install an ssh key on Windows? A module should be able to handle these use cases (or not handle them, but then they should exit gracefully).

#### Don't unit test your "roles" or "profiles"

This sounds counterintuitive, but you would never unit test a "main" function in a software program. That's absurd. Your "roles" or "profile" is a "main" class or an entrypoint to your infrastructure. This should not be unit tested (and if so, only so that it compiles). It should though have integration tests, system tests and the works. It's easy to get bogged down in unit testing here, but don't. It provides no value and that logic should have been already tested in your module.
