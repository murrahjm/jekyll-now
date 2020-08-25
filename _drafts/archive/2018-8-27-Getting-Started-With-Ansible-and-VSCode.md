---
layout: post
title: Getting Started with Ansible and VSCode
featured-img: ansible
categories: [Ansible]
---

This is the first post in a series of articles around using Ansible as a Windows Administrator managing, primarily, Windows machines.  However, many of the technologies I'll discuss are cross platform, so there should be plenty of info for everyone.  

[Part 1](/Getting-Started-With-Ansible-and-VSCode)

I'll start out by talking about how to use [VSCode](https://code.visualstudio.com/) and the [Windows Subsystem for Linux (WSL)](https://docs.microsoft.com/en-us/windows/wsl/install-win10) to create a great development and execution environment with Ansible.  Once we're all setup with our basic environment I'll go through some of the basics of Ansible and talk about the file layout and some terminology.

[Part 2](/Ansible-For-Windows-By-Windows)

Next I'll jump right in to managing Windows machines with Ansible.  I'll talk about the various connection methods, how to specify host and group-specific options, and how to encrypt our sensitive variables.  I'll then start on a basic playbook execution to test our connection, and talk about some of the modules available for configuring Windows hosts.

At this point you'll hopefully be comfortable with connecting to, and running playbooks against, your Windows hosts, so I'll take that model further by talking about some of the options for extending your playbooks with DSC resources and other, custom modules.

[Part 3](/Custom-Ansible-Modules-for-Fun-and-Profit)

And speaking of custom modules, we can create our own!  I'll talk about how you can create your own Ansible modules with PowerShell.  I'll talk about the basic structure of a module, some of the helper functions available, and how to structure the output.  I'll also talk about how to create modules that can run locally via PowerShell Core!

## Install and configure WSL

So let's get started!  Now I'm a Windows guy, always have been.  So I pretty much ignored Ansible for a while as a linux tool.  A coworker of mine, the 'linux guy' finally talked me into checking it out. In an effort to be more cross-platform friendly I finally relented.  He was super excited to show me around, and the first thing he did was ssh to a server and open up vi editor.
![VI editor, what more could you need?](/assets/img/posts/AnsibleForWindows/vim.png)
I learned two very important things from that discussion.

1. Linux guys *really* like text editors
2. Linux guys *really* don't like it when you laugh at their text editors

Social fumbles aside, I was ready to start trying out Ansible myself.  But I needed more than an SSH terminal and VI.  As cool as *"ctrl-g shift-O colon w q bang"* is, I just really needed some windows in my life.  Something like this would be nice:
![All the things!](/assets/img/posts/AnsibleForWindows/vscode-ansible.png)

### Install Ansible in WSL and dependent modules

## Configure VSCode

### Install Ansible extensions for VSCode

### Configure VSCode workspace to open bash as default shell

### Configure VSCode to use yaml as the default file type

## Create workspace

### Create new ansible folder

### Create basic subfolder structure

### create ansible.cfg file to use new structure

