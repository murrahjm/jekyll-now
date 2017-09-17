---
layout: post
title: Fun With Windows Firewall
---

Or maybe this is my actual first post.  First technical post anyway.  I did this thing a while back, and it turned out to be surprisingly useful despite it being an imperfect solution.  But I guess if it improves productivity and repeatability it can't be all bad.

So I'm not sure if the stars were aligned just right, or someone switched the decaf with regular somewhere, or if CIO magazine ran an article entitled "Security, why you need it and how to get it", but all of a sudden we had a group decision to enable host-based firewalls on all our Windows systems.  That's great!  But with a lack of any meaningful configuration management tools or processes, that's not so great.  The eager, naive me said "No problem, we'll use a GPO to enable the firewall and set rules for our most common traffic patterns, then the application and dev teams can handle creating local rules for their application traffic."  Oh sweet child, how nice that would be.  In the real world that translates into phone call after phone call of either "Can you tell me what ports my app uses and make those rules for me?", or more often, "You did something to my server and it's broken now OMG fix it!".  There was the occasionally pleasant "Here's the list of ports according to the vendor, please make rules".

All those boil down to the same thing, I'm creating local rules on servers one at a time.  [Powershell](https://technet.microsoft.com/en-us/library/jj554906(v=wps.630).aspx) to the rescue!  Say what?  Windows Server 2012R2 only?  Well I've got a pile of old 2008 servers ([yeah I know](https://support.microsoft.com/en-us/lifecycle/search?alpha=windows%20server%202008)), so I guess I'm stuck with [netsh commands](https://technet.microsoft.com/en-us/library/dd734783(v=ws.10).aspx).  Ancient technology aside, reading the wall of text known as the firewall log is not advisable for long term eyeball health (or sanity).  So with those requirements in mind, we started with these three cmdlets:

* __Get-FirewallLog__ - get the text-based windows firewall log and parse it into an array of objects
* __Get-FirewallRules__ - get a list of the current local firewall rules already set on the server, and turn it into an array of objects instead of a wall of text.
* __Add-FirewallRule__ - add a new local firewall rule.  takes parameter input and builds the appropriate netsh command

So those three provided the basic functionality to get started.  Huge time saver over the manual equivalent of each of those tasks.  Now for our specific use case we were only focusing on inbound traffic, so some of the rules stuff is obscured as that is a given assumption.  The next set of cmdlets came about after we started getting the same sort of requests and having to do the same sort of work.  Since powershell is awesome and lets you easily build upon existing commands, we're able to make more unique cmdlets built upon the basic functionality we already created.

For instance, we started getting requests like "Hey I have this new server can you just copy over the rules from my old server?"  Now even with the existing get-firewallrules and add-firewallrule cmdlets that's kind of a PITA.  So we build upon those to make:

* __Compare-FirewallRules__ - Gets the existing local firewall rules from two servers, and outputs any rules from server1 that aren't already on server2.
* __Copy-FirewallRules__ - Similar to Compare-FirewallRules, but takes any missing rules and creates them on server2

After a few troubleshooting instances of chasing down pesky programs with dynamic port allocations, we identified a common pattern.  Find the blocked port in the log, login to the server and use netstat to find the process that opened that port, then use a process listing to find the executable path for that process.  You do that more than a few times while someone is screaming on the phone about a broken application and you're looking to automate that in a hurry.  So we now have:

* __Get-ExecutableByPort__ - outputs the path to the executable that is responsible for the specified network port

And that's about it, a relatively useful set of tools that ended up saving a pretty rediculous amount of time over the equivalent manual processes.  So let me just wrap this part up by talking about the development flow here.  Notice how I separated the commands into those three groups, loosely based on creation time.  The thing with projects like these is that it should never really be "done".  It's like a cycle of improvement.  So you identify a need, create a tool to fit that need, and then you could just stop there.  But better to then start using those tools, and evaluate again for new needs.  New problems may present themselves because of the process change associated with the use of the new tools.  So you circle back after you're "done", and do the same thing; identify a need, create a tool to fit that need.  And just keep doing that, modifying and tweaking here and there until you're all out of needs and everyone is happy and starts throwing buckets of money at you and gives you a corner office and your own parking space, and...ok well maybe not all that but you get the idea.  And lest you dear reader think of me as some sort of process genius, I totally didn't come up with this idea.  On the Internets it appears to be referred to as the *Continuous Improvement Cycle*, and is illustrated by this powerpoint-worthy diagram:

![Just keep turning](https://mktgcdn.leankit.com/uploads/images/general/_large/Continuous_improvement_compressed@2x.jpg)

So this ended up being pretty big in its own right so let's break here and do a "Part 2" to start going over the details of the various commands.

Firewall Cmdlets index:
* Part 1(this post): Intro
* Part 2: Get-FirewallLog
* Part 3: Get-FirewallRules, Add-FirewallRules
* Part 4: Compare-FirewallRules, Copy-FirewallRules, Get-ExecutableByPort
