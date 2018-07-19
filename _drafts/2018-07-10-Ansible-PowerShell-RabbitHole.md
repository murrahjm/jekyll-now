---
layout: post
title: Ansible modules written in PowerShell for localhost
featured-img: ansible
categories: [Ansible]
---

This goes down the rabbit hole quite a bit, almost by accident.  Here's a stream of consciousness play by play of how I came to this strange place:

* Problem: I need to get some info about a vmware guest into a playbook
* Solution: Use a built-in Ansible module for vmware guest info

* Problem: The built-in Ansible module for vmware guest info doesn't give me exactly what I need
* Solution: Write a new module that works the way I need it

* Problem:  I don't know Python and am not familiar with the pyvmomi python module
* Solution: Use PowerShell instead


It's more complicated than that but that's the gist of it.  So I want to use the PowerCLI PowerShell module inside PowerShell Core, inside Ansible.  Turns out it works better than you might think!

Step 1:  Install PowerShell Core where you're running ansible
* works in WSL
* works in CentOS (machine running tower)

ansible modules use a she-bang at the top of the script file to tell the system what interpreter to use.  You can specify the direct path to the pwsh binary, and that will work.  But, if you want to use the helper files (Ansible.ModuleUtils.Legacy) you need a symlink. That helper module uses `#!powershell`, and since it ships with ansible you probably shouldn't modify it.  Also to be compatible with powershell modules run on remote windows systems, we should create a symlink for /bin/powershell to our actual pwsh binary path (this can vary)

powershell core ansible module works just like a powershell ansible module.  You can use the builtin helpers parse-args and  get-ansibleparam to turn the args input blob into powershell variables

Then you can use either exit-json to return structured output, or fail-json to return a structured error.  Everything in the middle is whatever you want.  We want to do some vmware stuff, so we need the PowerCLI module.  We get that on our box with `Install-module`, same as usual. At the time of this writing not every PowerCLI submodule run on linux.  Check out this article on how to modify the module manifest so that the module will import correctly.

https://cloudhat.eu/powercli-10-0-0-linux-error-vmware-vimautomation-srm/

test that it imports correctly and try running some commands.

In our module we probably already have this line at the top:

```powershell
#Requires -Module Ansible.ModuleUtils.Legacy
```

Since we need our PowerCLI module now too, we'll add a line for that:

```powershell
#Requires -Module Ansible.ModuleUtils.Legacy
#Requires -Module VMWare.PowerCLI
```

reference your module the same way you would any other module in a playbook.  Just put your module_name.ps1 file in a library folder and specify that in your ansible.cfg, or put it in the main library folder. (Check /etc/ansible/ansible.cfg for the location of the library folder)

if you want to use this powershell module with ansible Tower, you have to turn off Job isolation.

the way ansible works behind the scenes with powershell is that it creates a temporary copy of your module.ps1 file in a temp folder, then a temporary copy of all the passed arguments as a file named *args* in the same temp folder.  It then calls a command line something like this:

```bash
EXEC /bin/sh -c 'powershell /var/lib/awx/.ansible/tmp/ansible-tmp-1531236453.52-58852773538528/powershell_test.ps1 /var/lib/awx/.ansible/tmp/ansible-tmp-1531236453.52-58852773538528/args && sleep 0'
```

Job isolation redirects this to a temporary location on /tmp, so /var/lib/awx is really /tmp/<someuniquefoldername>.  It's not a symlink or anything though, just some tower magic.  But if your playbook has your powershell module-based task as something other than the first task in the playbook, then that temp folder will be gone when powershell tries to read that args file and will throw a gnarly error:

```
The specified drive root \"tmpfs\" either does not exist, or it is not a folder.\nFailFast:\nUnable to retrieve the specified information about the process or thread.  It may have exited or may be privileged.\n\n   at System.Environment.FailFast(System.String, System.Exception)\n   at System.Environment.FailFast(System.String, System.Exception)\n   at Microsoft.PowerShell.UnmanagedPSEntry.Start(System.String, System.String[], Int32)\n   at Microsoft.PowerShell.ManagedPSEntry.Main(System.String[])\n\nException details:\nSystem.ComponentModel.Win32Exception: Unable to retrieve the specified information about the process or thread.  It may have exited or may be privileged.\n   at System.Diagnostics.Process.GetStat()\n   at System.Diagnostics.Process.get_StartTimeCore()\n   at System.Diagnostics.Process.get_StartTime()\n   at Microsoft.PowerShell.ConsoleHost.DoCreateRunspace(String initialCommand, Boolean skipProfiles, Boolean staMode, String configurationName, Collection`1 initialCommandArgs)\n   at Microsoft.PowerShell.ConsoleHost.CreateRunspace(Object runspaceCreationArgs)\n   at Microsoft.PowerShell.ConsoleHost.DoRunspaceLoop(String initialCommand, Boolean skipProfiles, Collection`1 initialCommandArgs, Boolean staMode, String configurationName)\n   at Microsoft.PowerShell.ConsoleHost.Run(CommandLineParameterParser cpp, Boolean isPrestartWarned)\n   at Microsoft.PowerShell.ConsoleHost.Start(String bannerText, String helpText, String[] args)\n   at Microsoft.PowerShell.ConsoleShell.Start(String bannerText, String helpText, String[] args)\n   at Microsoft.PowerShell.UnmanagedPSEntry.Start(String consoleFilePath, String[] args, Int32 argc)\n/bin/sh: line 1:    38 Aborted                 powershell /var/lib/awx/.ansible/tmp/ansible-tmp-1531233049.46-78100593509002/powershell_test.ps1 /var/lib/awx/.ansible/tmp/ansible-tmp-1531233049.46-78100593509002/args\n",

```