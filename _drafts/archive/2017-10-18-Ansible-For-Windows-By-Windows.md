---
layout: post
title: Managing Windows Machines with Ansible
featured-img: ansible
categories: [Ansible]
---
I have a confession.  As much as I love PowerShell, I've been seeing another automation engine on the side.  I didn't expect it to happen, but it's so new and exciting, I just got carried away.  But my thoughts never strayed too far, and I started wondering how I can combine both of these loves into one joyous union.  So that's what I did!  And now I'm back to share because this combination feels like one of those giant swiss army knives with 100 tools.

![PowerShell + Ansible](../assets/img/posts/AnsibleForWindows/Giant_Knife.jpg)

Yeah, like that.  My intention is to do a series of blog posts covering all the interaction between Ansible and PowerShell, focusing primarily on managing Windows servers.  So this might be post number 1 or post number 3 eventually, but for now it'll be number one.  And I want to start number one with info you can use today.  How to connect and manage a Windows machine from Ansible.  Technology changes constantly, and Ansible being open source means it changes even faster.  For this particular article I'm using Ansible version 2.5.  I started playing with Ansible around version 2.2 I think, and things have improved dramatically in regard to management of Windows, so things are definitely headed in the right direction.

Before we get started, the thing to remember with Ansible is that when it performs work on a machine, it makes a remote connection to that machine and runs everything via that remote connection.  For linux this means SSH, but for Windows, this means a WinRM connection.  The same one that you use when you do `Enter-PSSession` or `Invoke-Command`.  Although if you're picturing it in your head, what Ansible does is more like `Enter-PSSession` than `Invoke-Command`.  This is in contrast to something like DSC which is, effectively, doing everything locally.

Therefore, if you have WinRM access to the machine, and a valid set of credentials, you're pretty much set to use Ansible to manage the machine.  "Pizza cake" as I like to say.  It wasn't always "Pizza cake" though. Previously, WinRM connection from ansible required either fully un-encrypted traffic on port 5985, or a valid SSL certificate to use with port 5986. Neither of these are supported by default on Windows, so before you can use ansible you have to configure all your hosts. Depending on the size of your environment that can range from annoying to impossible. However, once you pick a method (hint: don't do the un-encrypted traffic thing) and configure your machines, your connection variables in Ansible might look something like this:

```yaml
ansible_port: 5986
ansible_winrm_transport: kerberos
ansible_winrm_server_cert_validation: ignore
ansible_connection: winrm
```

You can check out [this article on WinRM](/WinRM-listeners) for more details, but the default settings for a Windows server is to listen on port 5985, but use [message encryption](https://docs.microsoft.com/en-us/powershell/scripting/setup/winrmsecurity?view=powershell-6). Looking through the latest ansible documentation, however, shows an option to enable message encryption, separate from any transport security.  If that works, then you should be able to do something like this:

```yaml
ansible_port: 5985
ansible_winrm_transport: kerberos
ansible_winrm_message_encryption: auto
ansible_connection: winrm
```

That would match the default settings on windows server.  Does it work?  Well no, not exactly.  Running a task with those connection settings gives me the following error:

```bash
 [WARNING]: ansible_winrm_message_encryption unsupported by pywinrm (is an up-
to-date version of pywinrm installed?)
```

Not up-to-date huh?  Well I can fix that. After updating pywinrm and some other modules, it worked!  I can now manage any windows server out of the box as long as winrm is turned on and I have credentials.  For the record, here are the versions of the modules that I needed to update to:

| module | version |
---------|----------
pywinrm | 0.3.0
pykerberos | 1.2.1
requests-kerberos | 0.12.0
requests-credssp | 1.0.0

Note that if you're not doing any credssp authentication then you probably don't need to install `requests-credssp`

Also Note that I did all this with ansible 2.5.  Depending on when you are reading this you may already have support for all this and the correct versions.



## Ansible Tower

To update the python modules for Tower, you have to perform the update in the correct virtual environment.  These are the commands.

```bash
source /var/lib/awx/venv/ansible/bin/activate
umask 0022
pip install --upgrade pywinrm
pip install --upgrade pykerberos
pip install --upgrade requests-kerberos
pip install --upgrade requests-credssp
deactivate
```