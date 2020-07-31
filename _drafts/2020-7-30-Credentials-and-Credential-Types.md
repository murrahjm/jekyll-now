# Ansible Tower Credentials

Where ansible provides `ansible-vault` for encrypting passwords and other secret strings, this method doesn't scale well with multiple playbooks and projects.
It also doesn't provide the granular access that Tower uses for most other objects.
The [credential](https://docs.ansible.com/ansible-tower/latest/html/userguide/credentials.html) object type in Tower provdes a more robust method for storing these secrets.
At the most basic level it provides the `machine` credential, which effectively passes the username and password to a machine when running a playbook.
This can be an ssh key if linux, or a username and password if using Windows.
It also provides options for specifying the become parameters for elevation.
There are a few benefits to this method:

1. **scalability** - when using an account in multiple playbooks, the ansible-vault encrypted string must be saved in each project folder for access.
When the time comes to change that password, it must also be updated in each of those saved locations.
By contrast, a credential object in Tower is created once, then can be applied to as many templates as needed, within a given organization.
When the password needs to be changed it only need to be updated in a single location, and no ansible code needs to be modified.
2. **granular control** - ansible-vault encryption is an all-or-nothing event.  The string used to perform the encryption is also used to decrypt and use the password.
If an account needs to be shared for use, the vault encryption string must be given to each party.
This also allows others to decrypt the string and retrieve the plaintext password.
Tower, on the other hand, provides granular access when sharing credentials.
Rights can be granted for `read`, `use`, `update`, and `admin`.
If needed, a credential can be shared with `use` rights, allowing someone to use the credential in a playbook, but not retreive the underlying plain text password.
Additionally, a credential object can be attached to a job template, and the job template itself can be shared.
In this scenario, the credential can only be used in the context of the job template, with no other use cases.
3. **modularity** - ansible-vault encrypted strings must live with the application code.
When moving from a development environment to production, replacing the credentials in an an automated fashion is tricky, often resulting in more encrypted secrets.
With Tower, however, the credential details are separated from the application code.
A credential object in the development Tower instance can reference the development user account, while the production instance can have a reference to the production account.
The code only needs a reference to the credential object, which can be modified in a pipeline, or the credentials could have the same name between environments.

# Creating a Credential

To create a credential object, login to the tower instance, navigate to the **CREDENTIALS** section, and click the green `+` sign.

![Add a credential](../../.attachments/Tower-credentials-new.png)

Give your credential object a recognizable name, and select the type from the **CREDENTIAL TYPE** field.
The credential type indicates how the credential will be used.
In the most common case, that of the `Machine` credential, the values are passed in to the playbook as the `ansible_user` and `ansible_password` variables.
This is the most common credential type, and is required when connecting to machines from an inventory file.

For other credential types, it is important to understand how the values are to be used in a playbook.

When you select a credential type, additional fields will appear on the form that are specific to that type.
In the case of the `Machine` credential, the fields support username / password pairs, as well as SSH key data.
In the case of a machine credential for an Active Directory (AD) account, use the following fields:

* **USERNAME** - the fully qualified account name.
Note the domain portion should be upper case for compatibility with linux KERBEROS requirements.
* **PASSWORD** - the password to the account.
Review [this](../Cyberark-Credential-Lookup.md) article for info on how to integrate with CyberArk.
* **PRIVILEGE ESCALATION METHOD** - for windows this will always be `runas`.
* **PRIVILEGE ESCALATION USERNAME** - in most cases this will be the same value as the **USERNAME** field.
* **PRIVILEGE ESCALATION PASSWORD** - the password for the above specified account.
Also supports CyberArk integration.

