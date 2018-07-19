---
layout: post
title: Executing DSC resources from Ansible with the win_dsc Module
featured-img: ansible
categories: [Ansible]
---


the win_dsc module mostly does a lot of parameter manipulation to turn the input hashtable into a format that can be passed to `Invoke-DSCResource' via splatting.  The help document for Invoke-DSCResource says 

> Before you run this cmdlet, set the refresh mode of the Local Configuration Manager (LCM) to Disabled.

The default value for a windows machine running PS version 5.1 is 'Push'.  This will result in the following error message from Ansible when trying to call the win_dsc module:

>"msg": "The data source could not process the filter. The filter might be missing or it might be invalid. Change the filter and try the request again.  ", "reboot_required": false

That error message is super unhelpful, and while it could be caused by other things, it is definitely caused by an incorrect `RefreshMode` value.  Use the below code to set it to Disabled.

```PowerShell
[DSCLocalConfigurationManager()]
configuration LCMConfig
{
    Node localhost
    {
        Settings
        {
            RefreshMode = 'Disabled'
        }
    }
}
lcmconfig
Set-DscLocalConfigurationManager -Path lcmconfig
```