#Create a Release Pipeline to Publish PowerShell Modules

You can't go anywhere these days without reading about devops, configuration as code, release pipelines, and all that other developer stuff.  For good reason of course, it's all pretty awesome stuff.  The latest promise of a path to that mythical golden land of IT, where everything just works and there are no midnight telephone calls.  But not everyone works in a customer-facing, developer-oriented business.  If half of your apps are 10 years old and your business practices are even older, it may seem impossible to make that transition.  And it might actually be impossible, since real change has to come from the top down.  But don't lose hope!  You can at the very least learn the process for your own purposes, and even get some sweet automation out of it as a bonus.

This post will cover one example of using source control and a release pipeline to go from a folder full of powershell scripts to a central repository with published powershell modules.  Even without any fancy devops-ian practices this is a good practice for anyone who has a collection of functions they are looking to share with their coworkers.  Add in the magic sauce and you get a repeatable, automatic process that keeps you in the coding game and out of the moving-files-around game.

## Step 1: Source Control

* myscript.ps1
* myscript.bak
* myscript.bakbak
* myscript.bak2
* myscript.txt

Now of course your script folder doesn't look like that, but I bet you know a guy who has a folder like that right?  The usefulness of those backups is n-1 seconds, n being your attention span or change window, whichever is longer.  "last write time" isn't all that reliable and no one goes in and puts any comments in anything, let's be honest.

There's some sort of immutable law of humans that the only effective way to make someone do something is to take away any more desirable options.  Or make it the only option.  If you give someone 5 ice cream flavors to choose from who knows what they'll pick.  But give them ice cream or brussel sprouts, bingo. (yes even if they're pan fried with pancetta and a balsamic reduction, ice cream still wins).

A source control system is that easiest path.  You put everything in one central location on your local drive because it's easy.  You add a commit message whenever you save a change because you have to.  You sync those changes up to the server because that's the only way to kick-off the automated deployment, which is way easier than a manual deployment.  Source control isn't awesome because it's the thing you should be using, it's awesome because it's the thing you want to use.  The thing that makes everything easier.  So let's take a look already!

### GIT

There are a few different source control systems out there, but GIT seems to be the most popular at the moment.  And it's what github uses, and we should all be putting code out on github and contributing to the community and all that right?  Ok so here's the world's shortest git demo.  [Github](https://try.github.io) has a pretty good git tutorial and there are tons of other resources out there as well.

You'll need a client, this is usually the git command line tool or git GUI if you're on Windows, and probably something on mac that costs more than it should.  And if you're running linux then you don't need me to tell you about how GIT works.  Anyway, go get the client from [here](https://git-scm.com/downloads) or for bonus points use find-package and install-package to install it.


##Step 2: Create the Repository Server

##Step 3: Create the Build, Test and Deploy Scripts
