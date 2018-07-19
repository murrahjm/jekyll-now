---
layout: post
title: Windows Remote Management
comments: true
categories: [PowerShell]
---
It occurred to me the other day that besides being useful for other folks to read, a blog could be useful for me to record stuff that I'm always forgetting or having to look up.  And one of those things is WinRM.  If you're not familiar, WinRM is a remoting protocol for Windows.  It's like SSH but useful (that's probably going to get me in trouble).  But as awesome as it is, it's not just a remoting protocol.  There's a whole thing underneath it to make it work.  Like an iceberg, if the iceberg were software.  A software defined iceberg maybe.  Anyway, so this blog post is going to go into the details of what WinRM is, how it works, what the various options are to it, and all that.  "But wait, I've already tried to read the technet articles on WinRM."  Ok yeah we're not going to go into that much detail.  Those links are included at the end if you want to get exhaustive info.  I'm going to try to distill it down into something you can read and understand in one cup of coffee or less.  Also, for my own selfish purposes this blog will serve as a single point for all these WinRM commands and settings that I keep forgetting.  So let's do it!

In a nutshell here's what you need to know.  WinRM is an implementation of the WSMAN specification.  That's only important because you'll see the word WSMAN in various places.  So for our purposes WSMAN = WinRM.  Outside the hard candy shell WSMAN (winrm) is basically a web server.  And just like a web server it has a handful of basic needs.

* A port to listen on for incoming connections
* A configured authentication protocol or protocols for incoming connections
* Other application-specific configuration things
* A configuration interface for setting all the above things

So in a nutshell that's a basic list of what a web server needs to be a useful web server.  And it's what WinRM needs to be a useful web-based management interface (web server).  So let's go through those.

# Listeners

In IIS you might call these bindings, but whatever you call them you have to get on the network and give yourself a port for incoming connections.  WinRM has default ports of 5985 and 5986, for HTTP and HTTPS respectively.  You can change those ports if you want, but you probably don't want to.

But wait there's more!  WinRM isn't just a web server, it's also a client (something, something, hair club).  That web server analogy only goes so far, I mean it's not like you're going to point your netscape browser at it.  Kids still use Netscape right?  Anyway, so the WinRM client settings mostly align with the WinRM server settings.  That's not really a big deal with listeners since the defaults all match up.  Unless of course you change all the port numbers around (again you probably don't want to do that).

By default WinRM is like my 4 year old, it doesn't listen to anything.  But unlike my 4 year old getting it to listen to requests is pretty simple.

```PowerShell
# create a listener on all IP addresses using HTTP transport
# The hard way:
winrm create winrm/config/listener?Address=*+Transport=HTTP
netsh advfirewall firewall add rule name="WinRM HTTP" dir=in protocol=tcp localport=5985 action=allow
# The easy way:
winrm quickconfig
```

So that works, but what if instead of talking to your 4 year old you want to talk to your wife about all the ice cream you're going to eat when the kids are asleep.  Then you're either spelling things or doing pig latin or something equally rediculous.  That's where HTTPS comes in (kids are worse than hackers, seriously).  That's a little more involved as you need a certificate on the host to use when enabling a listener.  But other than that you'd do it like this:

```PowerShell
# create a listener using HTTPS transport
# The hard way:
# First go find out the thumbprint of your server certificate
# First before that go get a certificate issued from an internal PKI or something
# First before that go get an internal PKI
winrm create winrm/config/listener?Address=*+Transport=HTTPS @{Hostname="ServerFQDN";CertificateThumbprint="97A2251B175DF6A2ABCB85FDF2FFF915DAB8"}
# The easy way:
# OK fine just create a self-signed cert with makecert.exe, but Jesus kicks a puppy every time you do that.
winrm quickconfig -Transport:HTTPS
```

So did that work?  Well let's check and see what listeners we have configured

```cmd
winrm enumerate winrm/config/Listener

Listener
    Address = *
    Transport = HTTP
    Port = 5985
    Hostname
    Enabled = true
    URLPrefix = wsman
    CertificateThumbprint
    ListeningOn = 10.1.2.3

Listener
    Address = *
    Transport = HTTPS
    Port = 5986
    Hostname = Server1.domain.com
    Enabled = true
    URLPrefix = wsman
    CertificateThumbprint = 97 A2 25 1B 17 5D F6 A2 AB CB 85 FD F2 FF F9 15 DA B8
    ListeningOn = 10.1.2.3
```

Oh and on the off chance you want to remove a listener:

```PowerShell
winrm delete winrm/config/listener?Address=*+Transport=HTTPS
# or:
winrm delete winrm/config/listener?Address=*+Transport=HTTP
```

Now that we have our listeners configured we just need to connect to the server.  But we don't want just anyone to be able to connect, so let's setup some authentication.

# Authentication

So this is the part where I think I'm supposed to post up some huge chart with like 15 different protocols and cross reference with like sunspots or something.  And then you either glaze over or run off to youtube.  So let's keep it simple.

* Basic = bad
* Kerberos = good
* Certificate = what are you the NSA?

Basic authentication for winrm is just like basic authentication on web servers, username and password flying free and unencumbered.  Kerberos is the preferred choice and should work for enterprise (domain joined) machines.  But for non-domain joined machines you're going to fall back to "negotiate" (NTLM).  It's better than basic but it ain't great.  Certificate authentication improves on negotiate for non-domain joined machines but it's certificates, so it's more work to setup.

The default settings are pretty good, kerberos and negotiate are both enabled, with kerberos being the first choice.  Note that by default the client settings are wide open, non judgemental, accept everyone.  There's another setting called "AllowUnencrypted" that must be changed to "True" in order for basic auth to work, but you know sometimes two locks on a door is better than one. So it doesn't hurt to just disable basic auth on the client all together.  To do this we'll go back to our trusty winrm command.

```cmd
winrm set winrm/config/Client/Auth @{Basic="False"}
```

__Tip__:
> WinRM the command line tool uses negotiate authentication to run commands even locally.  If you disable negotiate auth and kerberos auth you will break WinRM and lock yourself out.  You can fix this via the registry at HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\WSMAN\Service.  set auth_negotiate and auth_kerberos from 0 back to 1.

## CredSSP

While the above authentication protocols are more or less various degrees of secure, this guy does a unique thing to solve a particular problem.  So I think he deserves his own section in the alphabet soup of authentication protocols.  In short, CredSSP solves the issue with double-hop authentication.  Well "solves" is a strong word, but it's something.  Check the links at the bottom for more reading about all this but in a nutshell "double-hop" authentication is when you connect from machine A to machine B and you then need machine B to connect to machine C on your behalf.  Maybe you need a sql server to go connect to another sql server to access something that only you have access to.  Your password isn't some sweet sticky weed (whatever that is) to pass around to all your buddies, it's supposed to be a secret between machine A and machine B.  Kerberos has a whole delegation model where you specify ahead of time which services can pass your credentials around and what they can use them for.  CredSSP just says "eh you look like a good guy so I'm just going to put my username and password inside this envelope.  You use them if you need to but keep them secret. or something."

Now I'm not a security blogger so I won't get into a debate about credssp vs kerberos constrained delegation.  But if you need CredSSP for WinRM, you can enable it like this:

```PowerShell
#run this on the machine you're going to connect FROM
Enable-WSManCredSSP -Role "Client" -DelegateComputer "Server1.domain.com" #ideally you restrict this to just a single machine you're connecting to.  You can use wildcards but see above warning about Jesus and puppies
#run this on the machine you're connecting TO
Enable-WSManCredSSP -Role "Server"
```

That will allow the "client" to send its username and password over to the "server", and for the "server" to use those credentials however it wants. Read that last part again.  So yeah not a super secure thing, and you have to really trust the server you send your creds to.  So only do that if you really need CredSSP and are aware of all the security things.  Like when your dad bought you a BB gun but made you read the whole friggen safety manual before you could even touch the thing.

# Configuration

So authentication and encryption and all that is great!  And by now you should be up and running and in remote heaven.  But if you're wondering about all those code samples and where that's all coming from, then let's talk about that.  To go back to our web server analogy, we might store all of our configuration settings in a file, like a web.config or something.  Well WinRM, being a web server, does a similar thing.  Not exactly a file per se, but a "WS-Management catalog".  But you can look at it like a file with the configuration tool WinRM (not to be confused with the service or server or client or feature).  Fun fact, WinRM is actually a cmd file that calls a vbscript, that handles all the interaction with that service catalog.  Probably the easiest thing to do it just run a command that will output all the current WinRM configuration settings.  This would be the closest thing to looking at our web.config file.

```cmd
winrm get winrm/config

Config
    MaxEnvelopeSizekb = 500
    MaxTimeoutms = 60000
    MaxBatchItems = 32000
    MaxProviderRequests = 4294967295
    Client
        NetworkDelayms = 5000
        URLPrefix = wsman
        AllowUnencrypted = false
        Auth
            Basic = true
            Digest = true
            Kerberos = true
            Negotiate = true
            Certificate = true
            CredSSP = true
        DefaultPorts
            HTTP = 5985
            HTTPS = 5986
        TrustedHosts = *
    Service
        RootSDDL = O:NSG:BAD:P(A;;GA;;;BA)(A;;GR;;;IU)S:P(AU;FA;GA;;;WD)(AU;SA;GXGW;;;WD)
        MaxConcurrentOperations = 4294967295
        MaxConcurrentOperationsPerUser = 1500
        EnumerationTimeoutms = 240000
        MaxConnections = 300
        MaxPacketRetrievalTimeSeconds = 120
        AllowUnencrypted = false
        Auth
            Basic = false
            Kerberos = true
            Negotiate = true
            Certificate = false
            CredSSP = true
            CbtHardeningLevel = Relaxed
        DefaultPorts
            HTTP = 5985
            HTTPS = 5986
        IPv4Filter = * [Source="GPO"]
        IPv6Filter [Source="GPO"]
        EnableCompatibilityHttpListener = false
        EnableCompatibilityHttpsListener = false
        CertificateThumbprint = 82 97 a2 25 1b 17 5d f6  a2 ab cb 85 fd f2 ff f915 da b8
        AllowRemoteAccess = true [Source="GPO"]
    Winrs
        AllowRemoteShellAccess = true
        IdleTimeout = 7200000
        MaxConcurrentUsers = 10
        MaxShellRunTime = 2147483647
        MaxProcessesPerShell = 25
        MaxMemoryPerShellMB = 1024
        MaxShellsPerUser = 30
```

So that's a whole mess of stuff.  And it's not everything, but it's most of the things.  You can see there some of the things we've already talked about, the client authentication options, the server authentication options, some of the listener info.  You can see that "AllowUnencrypted" setting ([Don't change that](https://blogs.msdn.microsoft.com/PowerShell/2015/10/27/compromising-yourself-with-winrms-allowunencrypted-true/).  No not even if some [product documentation](https://pubs.vmware.com/orchestrator-plugins/index.jsp?topic=%2Fcom.vmware.using.PowerShell.plugin.doc_10%2FGUID-D4ACA4EF-D018-448A-866A-DECDDA5CC3C1.html) tells you to).  This view is great as it shows you the heirarchy you can use to build any command.  So if you want to change any value, you just use the format:  ```winrm set <path/to/setting> @{key=value}```.  If you want to get a particular value you can use the same format with ```winrm get```.

So that's a pretty good tool, and if one tool is good, then two tools is even better!  Enter PowerShell:

```PowerShell
# PowerShell mounts the WSMan catalog as a file system or 'PSDrive'
Get-PSDrive WSMan
# standard filesystem commands like Get-ChildItem (gci or dir), Set-Location(cd) work
gci wsman:\localhost
# You can change key-value pair settings with set-item
set-item WSMan:\localhost\MaxEnvelopeSizekb -value 500
```

# Other Settings

Now that we're familiar with the WinRM configuration and tools, let's look at some of the other things you're likely to have to change.  These all revolve around your remote sessions, and you might run into some of these limits when you start going crazy with remoting.  You can see these settings in the Winrs section of the winrm config output above, or just look at it here:

```cmd
    Winrs
        AllowRemoteShellAccess = true
        IdleTimeout = 7200000
        MaxConcurrentUsers = 10
        MaxShellRunTime = 2147483647
        MaxProcessesPerShell = 25
        MaxMemoryPerShellMB = 1024
        MaxShellsPerUser = 30
```

These are all server-side settings, and control things like how much memory a single session can use (```MaxMemoryPerShellMB```) or how many concurrent sessions you can have open at once (```MaxShellsPerUser```).  You can change any of these settings with the utilities listed above, and the same key-value pair format applies.

```cmd
winrm set winrm/config/winrs @{MaxMemoryPerShellMB="2048"}
```

or

```PowerShell
Set-item WSMan:\localhost\Shell\MaxShellsPerUser -Value 20
```

Note that the above examples are just syntax examples, and not any kind of recommendation for best practice values or anything.  The defaults here are generally pretty good, but you might at some point need to change these if you have some sort of unique workload.

# Done?

whew!  Ok so I may not have met my goal of keeping this to a one-cup of coffee post, but hopefully it's a bit more compressed and digestible than the below wall of links.  Pretty much everything I wrote about I looked up in the below URLs though, so if you want more details they should all be in there.  I would say if you have any questions to post them here, but I haven't figured out how to add comments to this blog yet so you can't.  And really Internet comments just make me lose faith in humanity as a whole, so probably best to avoid that anyway.

## Reference links

### WinRM

* [https://support.microsoft.com/en-us/help/2019527/how-to-configure-winrm-for-https](https://support.microsoft.com/en-us/help/2019527/how-to-configure-winrm-for-https)
* [https://msdn.microsoft.com/en-us/library/ee309365(v=vs.85).aspx](https://msdn.microsoft.com/en-us/library/ee309365(v=vs.85).aspx)
* [https://blogs.msdn.microsoft.com/PowerShell/2015/10/27/compromising-yourself-with-winrms-allowunencrypted-true/](https://blogs.msdn.microsoft.com/PowerShell/2015/10/27/compromising-yourself-with-winrms-allowunencrypted-true/)
* [https://msdn.microsoft.com/en-us/library/aa384372(v=vs.85).aspx](https://msdn.microsoft.com/en-us/library/aa384372(v=vs.85).aspx)
* [https://docs.microsoft.com/en-us/PowerShell/scripting/setup/winrmsecurity?view=PowerShell-5.1](https://docs.microsoft.com/en-us/PowerShell/scripting/setup/winrmsecurity?view=PowerShell-5.1)
* [https://msdn.microsoft.com/en-us/library/ee309367(v=vs.85).aspx](https://msdn.microsoft.com/en-us/library/ee309367(v=vs.85).aspx)
* [http://www.hurryupandwait.io/blog/certificate-password-less-based-authentication-in-winrm](http://www.hurryupandwait.io/blog/certificate-password-less-based-authentication-in-winrm)

### Double-Hop Authentication

* [https://blogs.technet.microsoft.com/askds/2008/06/13/understanding-kerberos-double-hop/](https://blogs.technet.microsoft.com/askds/2008/06/13/understanding-kerberos-double-hop/)
* [https://blogs.msdn.microsoft.com/besidethepoint/2010/05/08/double-hop-authentication-why-ntlm-fails-and-kerberos-works/](https://blogs.msdn.microsoft.com/besidethepoint/2010/05/08/double-hop-authentication-why-ntlm-fails-and-kerberos-works/)
