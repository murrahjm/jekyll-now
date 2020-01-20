---
layout: post
title: Managing Windows Machines with Ansible
featured-img: linux
categories: [WSL]
---
# Work is work and play is play, except for new laptop day.

I didn't actually intend to make a corny rhyme when I started writing but now that it's there I stand by it.
When hardware refresh cycles come around and a shiny new computer appears at work, it is a magical time!
The promised increase in performance overshadows the drudgery of reinstalling years worth of applications, settings, and files.

This is not an article about [chocolatey](https://chocolatey.org), but using [chocolatey](https://chocolatey.org) to reinstall all those windows apps is awesome and you should use it!
Check out [the chocolatey website](https://chocolatey.org) for more info.
And no, I'm not being compensated by [chocolatey](https://chocolatey.org) every time I mention the word [chocolatey](https://chocolatey.org), that's just crazy.

This is an article about the [Windows Subsystem for Linux (WSL)](https://docs.microsoft.com/en-us/windows/wsl/about).
WSL is equally as exciting and awesome as [chocolatey](https://chocolatey.org), but is otherwise unrelated.
Specifically this is an article about how to get your WSL environment from your old machine over to your new machine without destroying anything or pulling all your hair out.
If you're like me and have been using WSL for any length of time, you may have tons of customized files, installed packages, and other unique configurations that could be difficult or impossible to reproduce.
Luckily someone at Microsoft thought about that so they added an import/export function to WSL!
That is where our journey will start.
Of course if it was as easy as just running a command I wouldn't be bothering to write a blog post, so read ahead for all the exciting twists and turns!

## Backing up the old machine

First things first.
Before we do anything we need to get a backup of our current setup.
The good news is WSL (the command-line tool, not the technology) has an import/export feature.
The bad news is that it was added in Windows 10 version 1903.
Hopefully you're already at that version or a newer one.  If not you might have to make friends with whoever controls OS upgrades at your company.

Once your machine is at version 1903 (however that happened, no questions asked) you can run this command to create the backup file.
Note this example is using Ubuntu version 18.04.  Use `wsl.exe -l` to list your installed distributions and update the command as needed.

```cmd
wsl.exe --export Ubuntu-18.04 c:\temp\wsl-backup.tar
```

Once that completes you should have a very large file that contains all of your WSL data.
Use your favorite networking technology to transfer that file to your new machine.
And read on, because it's about to get ~~weird~~ interesting.

## Preparing the new machine

Importing the WSL configuration has the same requirements as exporting, namely that you're running Windows 10 version 1903.
Other than that there's a little prep to do before you can run your `wsl --import` command.

Why? Well you have a backup, but it turns out that is only the backup of your distribution.
While that's everything from inside the linux environment, there's more to WSL than just that.
Namely all the things that make it a windows installation, so things like the executables, paths, various libraries, etc.
In order to get all of that and still have your backed-up environment, you've got to do a little maneuvering.

## Maneuvering

1. The first step is to download a complete linux distribution package.  If you can't access the Microsoft Store you can download the ubuntu package from here:  https://aka.ms/wsl-ubuntu-1804.
If you want to use a different distribution, check out the list here: https://docs.microsoft.com/en-us/windows/wsl/install-manual.
This may download a zip file instead of an appx file.  If so, rename it to .appx.  Alternately, use this powershell to download it.

```powershell
Invoke-WebRequest -Uri https://aka.ms/wsl-ubuntu-1804 -OutFile Ubuntu.appx -UseBasicParsing
```

2. Run the appx file to install Ubuntu (or whatever distribution you downloaded).
Even if your work policy blocks access to the Microsoft store you should be able to install a downloaded file.
After the installation is complete it should launch and ask you for an initial username.
You can just close the window at this point, none of that setup is needed.

3. Now that the binaries and all the other necessary bits are installed, we have a fully functional linux environment.
We need to do a little trickery to get it updated to be *our* linux environment.
First, open file explorer (or a terminal window) and browse to the `%localappdata%\packages` folder and find the subfolder where the distribution was installed.  Mine was this:
`C:\Users\jeremy\AppData\Local\Packages\CanonicalGroupLimited.Ubuntu18.04onWindows_79rhkp1fndgsc`.
Confirm this is correct by opening the `LocalState` subfolder and verify there are `rootfs` and `temp` subfolders.
Note that this `rootfs` folder is your linux environment.  **NEVER TOUCH ANYTHING IN HERE FROM WINDOWS OR IT WILL ALL BREAK, THE ICE CAPS WILL MELT, THE SUN WILL GO SUPERNOVA, AND ALL THE BABY PENGUINS WILL DIE!!**

4. The WSL import process creates a distribution, but you can't do that while this new one exists.
Open terminal window and unregister the newly installed distribution: `wsl.exe --unregister Ubuntu-18.04`.
Note that while this removes the distribution, the Ubuntu appx package is still installed.
This can be verified by checking the start menu, which should still have an Ubuntu icon.
ALso, the `Localstate` folder from above should still exist and have a temp folder.
The `rootfs` folder is gone, but that kind of makes sense since we removed our distribution.

5.  Now that the distribution is gone you can finally run the import to inject your customized distribution.
Note in the command below that the folder for install must be the location of the original install above, specifically the `Localstate` subfolder.
The distribution name probably has to be the same too, though I didn't test that.

```cmd
wsl.exe --import Ubuntu-18.04 C:\Users\jeremy\AppData\Local\Packages\CanonicalGroupLimited.Ubuntu18.04onWindows_79rhkp1fndgsc\Localstate c:\users\jeremy\downloads\wsl-backup.tar
```

6. At this point the rootfs folder should get recreated as your data is extracted into it.
Run bash.exe (or wsl.exe), and a console window should open and login as root.
You can verify some of your files are there if you like.

7. The last step is to configure the default user for linux, so it opens as you instead of root. This can be done with the following command

```cmd
ubuntu1804.exe config --default-user jeremy
```

## Wrap-up

At this point you should be all set, everything should be running just as it was before.
On the one hand this is a surprisingly complicated process, but on the other hand the fact that it works at all is pretty great.
However if you're exhausted from all this effort, be sure to check out [chocolatey](https://chocolatey.org) to make the rest of your re-installation efforts as easy as possible.