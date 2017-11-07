#WinRM

WinRM is awesome.  seriously it's like the best thing since RDP.  If you're not familiar, WinRM is a remoting protocol for Windows.  It's like SSH but useful (that's probably going to get me in trouble).  But as awesome as it is, it's not just a remoting protocol.  There's a whole thing underneath it to make it work.  Like an iceberg, if the iceberg were software.  A software defined iceberg maybe.  Anyway, so this blog post is going to go into the details of what WinRM is, how it works, what the various options are to it, and all that.  "But wait, I've already tried to read the technet articles on WinRM."  Ok yeah we're not going to go into that much detail.  Those links are included at the end if you want to get exhaustive info.  I'm going to try to distill it down into something you can read and understand in one cup of coffee or less.  Then you can do something else for your second cup.

So here's what you need to know.  WinRM is an implementation of the WSMAN specification.  That's only important because you'll see the word WSMAN in various places.  So for our purposes WSMAN = WinRM.  Outside the hard candy shell WSMAN (winrm) is basically a web server.  And just like a web server it has a handful of basic needs.

* A port to listen on for incoming connections
* A configured authentication protocol or protocols for incoming connections
* A certificate for encryption and the option to use that (this is 2017, SSL all the things!)
* Other application-specific configuration things
* A configuration interface for setting all the above things

So in a nutshell that's a basic list of what a web server needs to be a useful web server.  And it's what WinRM needs to be a useful web-based management interface (web server).  So let's go through each of those.

## Listeners

In IIS you might call these bindings, but whatever you call them you have to get on the network and give yourself a port for incoming connections.  WinRM has default ports of 5985 and 5986, for HTTP and HTTPS respectively.  You can change those ports if you want, but you probably don't want to.

But wait there's more!  WinRM isn't just a web server, it's also a client (something, something, hair club).  That web server analogy only goes so far, I mean it's not like you're going to point your netscape browser at it.  Kids still use Netscape right?  Anyway, so most of the listener and authentication configuration is identical between server and client.  That's not really a big deal with listeners unless you change all the port numbers around (again you probably don't want to do that).

By default WinRM is like my 4 year old, it doesn't listen to anything.  But unlike my 4 year old getting it to listen to requests is pretty simple.

```cmd
REM create a listener on all IP addresses using HTTP transport
REM The hard way:
winrm create winrm/config/listener?Address=*+Transport=HTTP
netsh advfirewall firewall add rule name="WinRM HTTP" dir=in protocol=tcp localport=5985 action=allow
REM The easy way:
winrm quickconfig
```

So that works, but what if instead of talking to your 4 year old you want to talk to your spouse about all the ice cream you're going to eat when the kids are asleep.  Then you're either spelling things or doing pig latin or something.  So we'll just do HTTPS for that.  That's a little more involved as you need a certificate on the host to use when enabling a listener.  But other than that you'd do it like this:

```cmd
REM create a listener using HTTPS transport
REM The hard way:
REM First go find out the thumbprint of your server certificate
REM First before that go get a certificate issued from an internal PKI or something
REM First before that go get an internal PKI
winrm create winrm/config/listener?Address=*+Transport=HTTPS @{Hostname="ServerFQDN";CertificateThumbprint="97A2251B175DF6A2ABCB85FDF2FFF915DAB"}
REM The easy way:
REM OK fine just create a self-signed cert with makecert.exe, but Jesus kicks a puppy every time you do that.
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
```

Oh and on the off chance you want to remove a listener:

```cmd
winrm delete winrm/config/listener?Address=*+Transport=HTTPS
REM or:
winrm delete winrm/config/listener?Address=*+Transport=HTTP
```

## Authentication

So this is the part where I think I'm supposed to post up some huge chart with like 15 different protocols and cross reference with like sunspots or something.  And then you either glaze over or run off to youtube.  So let's keep it simple.

* Basic = bad
* Kerberos = good
* Certificate = what are you the NSA?

Basic authentication is just like on web servers, username and password flying free and unencumbered.  Kerberos is the preferred choice and should work for enterprise (domain joined) machines.  But for non-domain joined machines you're going to fall back to "negotiate" (NTLM).  It's better than basic but it ain't great.  Certificate authentication improves on negotiate for non-domain joined machines but it's certificates, so it's more work to setup.

The default settings are pretty good, kerberos and negotiate are both enabled, with kerberos being the first choice.  Note that by default the client settings are wide open, non judgemental, accept everyone.  There's another setting called "AllowUnencrypted" that must be changed to "True" in order for basic auth to work, but you know sometimes two locks on a door is better than one. So it doesn't hurt to just disable basic auth on the client all together.  To do this we'll go back to our trusty winrm command.

```cmd
winrm set winrm/config/Client/Auth @{Basic="False"}
```

__Tip__:
> WinRM the command line tool uses negotiate authentication to run commands even locally.  If you disable negotiate auth and kerberos auth you will break WinRM and lock yourself out.  You can fix this via the registry at HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\WSMAN\Service.  set auth_negotiate and auth_kerberos from 0 back to 1.

### CredSSP

While the above authentication protocols are more or less various degrees of secure, this guy does a unique thing to solve a particular problem.  So I think he deserves his own section in the alphabet soup of authentication protocols.  In short, CredSSP solves the issue with double-hop authentication.  Well "solves" is a strong word, but it's something.  Check the links at the bottom for more reading about all this but in a nutshell "double-hop" authentication is when you connect from machine A to machine B and you then need machine B to connect to machine C on your behalf.  Maybe you need a sql server to go connect to another sql server to access something that only you have access to.  Your password isn't some sweet sticky weed (whatever that is) to pass around to all your buddies, that was a secret between machine A and machine B.  Kerberos has a whole delegation model where you specify ahead of time which services can pass your credentials around and what they can use them for.  CredSSP just says "eh you look like a good guy I'm just going to put my username and password inside this envelope.  You use them if you need to but keep them secret. or something"

Now I'm not a security blogger so I won't get into a debate about credssp vs kerberos constrained delegation.  But if you need CredSSP for WinRM, you can enable it like this:

```cmd
  
```

## Reference links
### WinRM
* https://support.microsoft.com/en-us/help/2019527/how-to-configure-winrm-for-https
* https://msdn.microsoft.com/en-us/library/ee309365(v=vs.85).aspx
* https://blogs.msdn.microsoft.com/powershell/2015/10/27/compromising-yourself-with-winrms-allowunencrypted-true/
* https://msdn.microsoft.com/en-us/library/aa384372(v=vs.85).aspx
* https://docs.microsoft.com/en-us/powershell/scripting/setup/winrmsecurity?view=powershell-5.1
### Double-Hop Authentication
* https://blogs.technet.microsoft.com/askds/2008/06/13/understanding-kerberos-double-hop/
* https://blogs.msdn.microsoft.com/besidethepoint/2010/05/08/double-hop-authentication-why-ntlm-fails-and-kerberos-works/
### CredSSP
* 
* 