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

### Day 1

If you're one of those lucky people that have gone cloud-only, your Day 1 has probably gone the way of the telegram. For those of us stuck in the world of virtual machines, SSH, RDP, and tedious change orders you're stuck with operating a lot of shit and hopping through a spider web of bureaucracy. This "shit" piles up. Sharepoint. Exchange. Elasticsearch. Tomcat. Apache. An Oracle RAC cluster here. A hand full of MS SQL servers there. A splatter of IIS servers. Maybe an ESB. Maybe two? Possibly even three? These servers need to get provisioned and that expensive software isn't going to install itself. The pain point eventually becomes not having enough competent hands to take care of downstream demand, while maintaining a reasonable operational state. The digitalization journey is running at a faster pace than those hands can type. Something has to give. 

For us, we simply wanted a way to easily manage SSH keys and Tivoli Storage Manager configurations for a few hundred servers. TSM upgrades could take a few days. Configuration changes would also take a few days. Add those changes to the simple fact that we were always finding out several weeks later about an unknown server that never got updated. Additionally, We were spending an inordinate amount of time installing SSH keys to suppliers and/or developers. We started loosing control of our infrastructure. We really didn't have a clue who had access to what. Permissions started with a change and then continued forever. In the end, we sat down and simply asked, "Can't we do better than this?"

### Day 2

Configuration management tools approach these Day 1 problems by turning the problem on its head. Instead of competent hands typing away, performing ad hoc changes on servers based on downstream and upstream demand, the enterprise's fleet of servers suddenly become configureable artifacts. Just like an application, these artifacts can be patched, altered, released and tested. Those hands suddenly move from SSH and RDP as their toolset over to their favorite IDE. Instead of installing a packet with let's say:

```bash
ssh -l root foo.domain.com "apt-get install -y apache2"
```

Your Ops tech does something like:

```puppet
node 'foo.domain' {
    packet {'apache2':
      ensure => 'installed',
    }
}
```

This becomes kind of nice. This is an example of a simple package installation, but those pesky Tivoli Storage Manager upgrades and configurations that we had now take on a different look. We can create a simple in-house repository, package up the latest release to Debian packages (Tivoli Storage Manager previously only had RPM packages), and pop them in to our repo.

File `manifests/tsm.pp` looks something like this:

```puppet
apt::source {'my_repo':
  location => 'https://repo.domain.com',
  key      => {
      'id'     => '123456789',
      'server' => 'https://gpg.domain.com',
  }
}

package {'tsm':
  ensure => 'latest',
  require => Apt::Source['my_repo],
}
```

And then every Unix server that we had in our fleet, could have a `site.pp` file with something like:

```puppet
node default {
    include ::tsm
}
```

Wow! How much nicer is this? We can release a new TSM package and it's black magic! TSM just upgrades itself on the server! We don't miss a server, or we do... but then we simply add a puppet agent and then we never miss it again! We can toss in a configuration file for good measure by simply adding the following line to `manifests/tsm.pp`.

```puppet

file {'/opt/tivoli/conf/tsm.sys
    ensure => file,
    source => 'puppet:///modules/my_repo/tsm.configuration',
}
```

Now, everything that's in the file `tsm.configuration` just magically appears on **EVERY** server in our enterprise!

Looking back, it's hard to express how amazing my first step into the world of "Infrastructure as Code" really was. I had programmed enough to understand code, but back then "code" for me was a mess of Perl scripts lying around on my Ubuntu PC to do some kind of regex, text parsing. Some of these scripts were tossed into cronjobs. Maybe somebody wanted an Oracle database updated. Maybe someone wanted an email. Now though, with "Infrastructure as Code", our infrastructure could be manipulated and coded and altered and controlled. It's hard to express in words, but everything changed at this moment.

### Day 3

Day 3 applies and tests the waters that start in Day 2. After the eye-opening moment of discovering "Infrastructure as Code", there is a suddent realization that all of this code needs to get written. Everything that needs to get done for the first time is really easy... until it's not. Donald Rumsfeld once made the comment about the risks and problems of invading Iraq many years ago, speaking about the "unknown unknowns". In other words, you not only run into problems that you didn't assess, but rather no one even knows that a calculable issue even exists.

Personally, and I can't stress this enough, every beginning begins with uncalculable issues that you could never have imagined. Make time for these issue! They'll come. I promise. You can mitigate these issues by bringing in help or buying support. If you can, and you trust the help you're buying, you probably should. It helps with a lot of these issues. But sometimes you can't. Maybe there's no money in the bank. But maybe even more likely, this "Infrastructure as Code" thingy is probably very grassroot-ish and your team can't really formulate a business case. No enterprise in their right mind is going to drop a few hundred thousand on some TSM upgrades and SSH key deployments. It ain't happenin'. But the "unknown unknowns" cut both ways. Not only do you not only not know what issues you could potentially run into, you don't know what potential issues you're going to solve. No matter though how you twist and turn, you're business case is going to look shoddy! 

`- We have a fantastic idea that's going to solve a look of stuff.
`-  Like what?
`- We're not sure.
`- What kind of risks do you foresee?
`- No idea.

This is one of those cases where you could literally be laughed out of the board room.

In our experience, everything was from the grassroots. We installed a Puppet server on a test server and built a site.pp from scratch. We built our TSM configuration templates and made them as simple as we could. We created "node definitions" for every server and we were more or less content.

```puppet
node {'server1.domain.com':
  include ::tsm
  include ::unix_staff_ssh_keys
}

node {'server2.domain.com':
  include ::tsm
  include ::unix_staff_ssh_keys
  include ::wordpress_devs_ssh_keys
}

node {'server3.domain.com':
  include ::tsm
  include ::unix_staff_server_keys
  include ::java_devs_ssh_keys
}
```

### Day 4



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
