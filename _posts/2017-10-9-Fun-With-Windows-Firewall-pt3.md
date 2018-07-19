---
layout: post
title: Fun With Windows Firewall pt. 3
featured-img: windowsfirewall
categories: [PowerShell]
---
This post is part of a series of posts outlining a handful of windows firewall management cmdlets.  See the intro post [here](/Fun-With-Windows-Firewall).

TL;DR get the code [here](https://github.com/murrahjm/misc-scripts/tree/master/WindowsFirewallcommands)

# Get-FirewallRules

A quick word about the process.  Basically in all these cmdlets we find the way to get the data, the "hook", then we just add a bunch of window dressing to make it pretty and functional.  In this case we need to get a list of all the existing firewall rules on the server.  So that's our "hook".  then we just wrap that up and do all the usual powershell things like this:

1. Get the data from the target machine.
2. Turn the text output data into powershell objects.
3. Define parameters to gather everything we need to retrieve the data.
4. perform error handling on our connection to the remote machine.
5. output objects in a pipeline-friendly format (object stream).

So let's talk about this "hook".  Like our firewall log cmdlet we want this to be backward compatible to Windows Server 2008, so the latest built in cmdlets are no good to us.  A quick search turns up [this great article](https://blogs.technet.microsoft.com/heyscriptingguy/2010/07/03/hey-scripting-guy-weekend-scripter-how-to-retrieve-enabled-windows-firewall-rules/) by the scripting guy with exactly what we're looking for.  Turns out we can use [new-object](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/new-object) to reference com objects, and get our rules that way.  And hey, they're already objects instead of a wall of text!  So that work is done for us.  We're going to pretty it up a bit, replace some of the codes with words, but otherwise we'll just output that.

```powershell
        $return = $(New-object -comObject HNetCfg.FwPolicy2).rules
            Foreach ($Rule in $Return){
...
                    Protocol = $(
                        Switch ($Rule.Protocol){
                            '1'  {'ICMPv4'}
                            '2'  {'IGMP'}
                            '17' {'UDP'}
                            '6'  {'TCP'}
                            '58' {'ICMPv6'}
                            '47' {'GRE'}
                            '41' {'IPv6'}
                        }
                    )
...
                    Direction = $(
                        Switch ($Rule.Direction){
                            '1' {'In'}
                            '2' {'Out'}
                        }
                    )

```

So that's pretty simple really, but let's do that window dressing.  First thing is our remote connection.  we want to be able to point this command at a remote machine and get our output.  WinRM makes this trivial but we do need a computername and possibly a set of alternate credentials.  And we don't want to try connecting to a bogus computername, so we'll do a quick validation of the parameter value.  That gives us a parameter block that looks like this:

```powershell
    Param(
        [Parameter(Mandatory=$True,Position=1)]
        [ValidateScript({test-connection -computername $_ -count 1})]
        [String]$Computername,

        [PSCredential]$Credential

    )

```

Next we'll want to handle any potential issues connecting to the remote machine with grace and dignity, like the Queen. (no not [that one](https://en.wikipedia.org/wiki/RuPaul)).  So we'll attempt to create a new powershell session (with or without credentials) and wrap that in a try/catch to catch those pesky errors.

```powershell
        write-verbose "$FunctionName`:  Attempting WSMAN connection to $computername"
        Try {
            If ($Credential){
                $Session = New-PSSession -ComputerName $computername -Credential $Credential
            } else {
                $Session = New-PSSession -ComputerName $computername
            }
            If ($Session) {Write-verbose "Connected to: $computername via WSMAN"} else {Throw}
        } Catch {
            write-error "Unable to connect to $computername" -ea Continue
            return
        }
```

And that's about it, we get our com object through the remote session, iterate through each returned object replacing codes with words, then output them to the pipeline.  Next up, let's make some new rules!

# Add-FirewallRules

Ok, so we've turned on the firewall on our favorite business app server, and all hell has broken loose.  We need to fix that sucker quick!  We need an easy to use cmdlet to make a firewall rule that doesn't take forever to figure out.  We want to basically make something that's easier to remember than netsh commands and can be executed remotely.  Piece of cake right?

We just need a computer name, a port, protocol, rule descriptor... oh wait, what if we want to do a dynamic rule for a service?  Ok so we just need a computername, a service name, a rule descriptor... oh right and we could also just want to do a rule for a particular executable.  Ok so we need a computer name, a program path, a... oh we just want to be able to pass a pre-formed rule in from the pipeline?  Like maybe the output from Get-FirewallRule? Ok hmm, well let's do all those things!  We can use [Parameter Sets](https://msdn.microsoft.com/en-us/library/dd878348(v=vs.85).aspx) to group our input options.  That will keep our service names out of our port-based rules, and our ports out of our program-based rules.  voila!

```powershell
Function Add-FirewallRule {
    [cmdletbinding(DefaultParameterSetName='byPort',SupportsShouldProcess=$True,ConfirmImpact='High')]
    Param(
        [Parameter(Mandatory=$True,Position=1)]
        [ValidateScript({test-connection -computername $_ -count 1})]
        [String[]]$Computername,

        [PSCredential]$Credential,

        [Parameter(Mandatory=$True,ParameterSetName='byPort')]
        [String]$Name,

        [Parameter(Mandatory=$True,ParameterSetName='byPort')]
        [string[]]$Ports,

        [Parameter(Mandatory=$True,ParameterSetName='byService')]
        [String]$ServiceName,

        [Parameter(Mandatory=$True,ParameterSetName='byProgram')]
        [String]$ProgramPath,

        [Parameter(Mandatory=$True,ParameterSetName='byPort')]
        [ValidateSet('TCP','UDP')]
        [String]$Protocol='TCP',

        [Parameter(ParameterSetName='byInputObject')]
        [Object]$Rule
    )
```

That might need a little deciphering.  What we have is basically 4 "options" for our input, grouping the necessary parameters together so we don't have conflicting input.  For instance, if we want to make a rule for a service, we really only need the name of the service.  Likewise with a program-based rule, we really only need the path to the executable.  In the case of a static port based rule we need the protocol, port, a rule name, etc.  Finally if we're passing an object from another cmdlet we want to be able handle whatever is in that object without any other parameters.  So each of those four options are grouped into parameter sets, as indicated by the ```ParameterSetName``` decoration.  Note that parameters can be a member of more than one parameter set, or they can be a member of all parameter sets.  In the case of our computername and credential parameters, we need those those no matter what kind of rule we're making, so we'll leave off that decoration and let them default to being in every parameter set.

Now that we have our parameter sets, let's walk through each one and see what we're going to do with them.  Spoiler alert, we're basically just going to be using the input to craft a netsh command string, then execute it on the remote machine.

## byPort

In this first parameter set we're going to create a firewall rule for a static Port/Protocol combo.

```powershell
        'byPort' {
            $PortString = $ports -join ','
            $portstring = $Portstring.replace(' ','').replace(',,',',')
            $CMDString = "netsh advfirewall firewall add rule `
                            name=`"$Name`" `
                            localport=`"$portstring`" `
                            dir=in `
                            action=allow `
                            enable=yes `
                            profile=any `
                            protocol=$Protocol"
        }
```
We're basically just doing string manipulation here.  Our 'Ports' parameter allows for an array of values as well as a single entry, so we're going to join these together with a comma, do some cleanup of any extra spaces or extra commas, then add that string of ports to the larger command string.  note that we are statically providing values for direction, action, profile, etc.  In the particular use case these cmdlets were written for we were only concerned with inbound rules.  For simplicity we've left those out of the parameter list, though they could easily be parameterized at a later date.

## byProgram

Here we're building a command string just like above, only we're providing the program path rather than port descriptors.  To keep thing simple we are calculating the rule name based on the name of the executable, and also statically setting some of the other options.

```powershell
        'byProgram' {
            $ProgramName = split-path $programpath -Leaf
            $Name = $ProgramName
            $CMDString = "netsh advfirewall firewall add rule `
                            name=`"$programName`" `
                            program=`"$programpath`" `
                            dir=in `
                            action=allow `
                            enable=yes `
                            profile=any"
        }
```

## byService

Same as above but even simpler.  We simply take the service name, tack it into the command string and call it a day.

```powershell
        'byService' {
            $Name = $ServiceName
            $CMDString = "netsh advfirewall firewall add rule `
                            name=`"$ServiceName`" `
                            service=`"$ServiceName`" `
                            dir=in `
                            action=allow `
                            enable=yes `
                            profile=any"
        }
```

## byInputObject

This one is a little more complicated.  We have to make some assumptions here about the kind of input object we're going to receive.  It could be a rule for any of the above parameter sets, so it might have port or protocol properties, or it might just have a program path or a service.  In general though, we are going to assume that this object is coming from the Get-FirewallRules cmdlet, so it should be reasonably well structured.  So we're basically just going to iterate through every possible property for the input object.  If the property is set we'll add it to the command string and continue on.  At the end we should have a commadn that includes every provided value.

```powershell
        'byInputObject' {
            $CMDString = "netsh advfirewall firewall add rule "
            If ($Rule.Name){
                $CMDString += "name=`"$($Rule.Name)`" "
                $Name = $Rule.name
            }
            If ($Rule.Description){$cmdstring += "description=`"$($Rule.Description)`" "}
            If ($Rule.ApplicationName){$cmdstring += "program=`"$($Rule.ApplicationName)`" "}
            If ($Rule.ServiceName){$cmdstring += "service=`"$($Rule.ServiceName)`" "}
            If ($Rule.Protocol){$cmdstring += "protocol=`"$($Rule.Protocol)`" "}
            If ($Rule.LocalPorts){$cmdstring += "localport=`"$($Rule.LocalPorts)`" "}
            If ($Rule.RemotePorts){$cmdstring += "remoteport=`"$($Rule.RemotePorts)`" "}
            If ($Rule.Action){$cmdstring += "action=`"$($Rule.Action)`" "}
            If ($Rule.LocalAddresses){$cmdstring += "localip=`"$($Rule.LocalAddresses)`" "}
            If ($Rule.RemoteAddresses){$cmdstring += "remoteip=`"$($Rule.RemoteAddresses)`" "}
            $cmdstring += "direction=in enable=yes profile=any"
        }
```

## Ready...Fire...Aim

At this point we have our netsh command string all ready to go, through one of the above methods.  All we have to do now is create our remote session, turn our command string into a scriptblock (```[scriptblock]::Create($cmdstring)```) and shoot it through invoke-command.  Netsh will return a passive-aggressive 'Ok.' if all went well.

Next up we'll look at taking these two cmdlets and building on them to do some more logic-y stuff with them.  I mean we did object output and standard input for a reason right?

## Firewall Cmdlets index

* [Part 1: Intro](/Fun-With-Windows-Firewall)
* [Part 2: Get-FirewallLog](/Fun-With-Windows-Firewall-pt2)
* [Part 3: Get-FirewallRules, Add-FirewallRules](/Fun-With-Windows-Firewall-pt3)
* Part 4: Compare-FirewallRules, Copy-FirewallRules
* Part 5: Add-ServiceFirewallRules, Get-ExecutableByPort
