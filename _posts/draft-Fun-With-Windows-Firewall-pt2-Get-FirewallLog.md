
So let's start with the log file first and work our way back from there.

# Get-FirewallLog

The goal of this cmdlet is pretty straight forward:  turn this painful wall of text: 
![firewall.log](/images/firewalllogtext.png)
Into sweet object-oriented goodness:
![objects!](/images/firewalllogobjects.png)

To do this we use the amazing, and kind of scary, [ConvertFrom-String](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertfrom-string?view=powershell-5.1).  I have no idea how this thing does its magic, but I think there might be some actual voodoo in there somewhere.  Basically you create a template file that is a representative sample of the text you want to parse, and you mark it up with variable names for each element you want.  So in the case of our firewall log we take this:

```text
$FirewallLogTemplate = @'
Version: 1.5
Software: Microsoft Windows Firewall
Time Format: Local
Fields: date time action protocol src-ip dst-ip src-port dst-port size tcpflags tcpsyn tcpack tcpwin icmptype icmpcode info path

2016-05-05 07:30:07 DROP UDP 10.96.105.243 224.0.0.252 61782 5355} 0 - - - - - - - RECEIVE
2016-05-05 07:30:21 DROP TCP 10.98.32.233 10.96.105.191 60164 1434 40 - - - - - - - RECEIVE
2017-04-07 10:45:43 DROP ICMP 10.32.72.58 10.96.101.190 - - 84 - - - - 0 0 - RECEIVE
2017-04-07 10:45:43 DROP ICMP 10.32.72.58 10.96.101.190 - - 84 - - - - 0 0 - RECEIVE
'@
```
And turn it into this:
```powershell
$FirewallLogTemplate = @'
#Version: 1.5
#Software: Microsoft Windows Firewall
#Time Format: Local
#Fields: date time action protocol src-ip dst-ip src-port dst-port size tcpflags tcpsyn tcpack tcpwin icmptype icmpcode info path

{[datetime]TimeStamp*:2016-05-05 07:30:07} {Action:DROP} {Protocol:UDP} {SourceIP:10.96.105.243} {DestIP:224.0.0.252} {SourcePort:61782} {DestPort:5355} 0 - - - - - - - RECEIVE
{[datetime]TimeStamp*:2016-05-05 07:30:21} {Action:DROP} {Protocol:TCP} {SourceIP:10.98.32.233} {DestIP:10.96.105.191} {SourcePort:60164} {DestPort:1434} 40 - - - - - - - RECEIVE
{[datetime]TimeStamp*:2017-04-07 10:45:43} {Action:DROP} {Protocol:ICMP} {SourceIP:10.32.72.58} {DestIP:10.96.101.190} {SourcePort:-} {DestPort:-} 84 - - - - 0 0 - RECEIVE
{[datetime]TimeStamp*:2017-04-07 10:45:43} {Action:DROP} {Protocol:ICMP} {SourceIP:10.32.72.58} {DestIP:10.96.101.190} {SourcePort:-} {DestPort:-} 84 - - - - 0 0 - RECEIVE
'@
```

So lets walk through this marked up nonsense.  First, the firewall log header.  We don't care about the version or the field listing or any of that.  Since this is powershell we're dealing with we just comment out those lines:

```powershell
#Version: 1.5
#Software: Microsoft Windows Firewall
#Time Format: Local
#Fields: date time action protocol src-ip dst-ip src-port dst-port size tcpflags tcpsyn tcpack tcpwin icmptype icmpcode info path
```

Now we have the sample firewall log entries.  This is a simple tab separated list, so we just want to turn each line into an object with named properties.  So for each value on the line we encapsulate it and give it a name, using the format ```{variablename:value}```.  Again since this is powershell under the voodoo we can get fancy with our declarations like this:```{[variabletype]variablename:value}```.  So we mark up the first line and give each tab-separated value a property name.  When ConvertFrom-String runs it will give us an object with properties named TimeStamp, Action, Protocol, SourceIP, etc.

So that's pretty cool, but what happens when we get to the second line?  We're going to have a bunch of duplicate properties when what we actually want is a second object in the output stream.  So we use the '*' designation on the variable name to indicate that this should be the beginning of a new object.  And that's about it.  When we run ConvertFrom-String and give it both the template and the file to process, it'll do its dark magic and spit out an array of glorious objects.  You can then do sorts, filters, csv outputs, etc.  Thanks Powershell!

We could stop there and have a pretty cool thing, but lets take it a small step further for better functionality.  Ideally we could have a cmdlet that will take just a computer name and a credential object and return the firewall log data.  So to do that we need a way to get the firewall log from the remote machine for processing.  Paging [Invoke-Command](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/invoke-command?view=powershell-5.1), you have a call on line 1.  PS remoting is generally pretty firewall friendly, and we have a computer name and credential object so it should be a trivial thing to run a get-content and pull the file contents over the remote session.  Wow is that the slowest thing ever or what!?  So ok maybe we should use SMB to just get the whole file locally and process it rather than waiting minutes retrieving the content

***put in measure-command code to show how slow it is

But SMB doesn't always work across various connections and firewalls and stuff, so let's have both.  We'll try SMB first, if that doesn't work we'll resort to the slower Invoke-command.  And for grins lets also make the connection method a user selectable parameter.
```powershell
    Param(
    ...
        [ValidateSet('SMB','WSMAN')]
        [String]$ConnectionMethod='SMB'
    )
```

To wrap it up we just need to handle connection errors from the file retrieval ([Try/Catch](https://blogs.technet.microsoft.com/heyscriptingguy/2014/07/05/weekend-scripter-using-try-catch-finally-blocks-for-powershell-error-handling/) works great for that), then handle some of the other file output options.
If the computer has the firewall disabled the file won't exist.
```powershell
        if ($Return) {
            If ($Return.length -le 7){
                Write-Verbose "$FunctionName`:  Firewall log found, but empty.  No blocked traffic reported"
            } else {
```
If the firewall is enabled but no blocked traffic has been logged yet, you'll get a file with header info but nothing else.
```powershell
        } else {
            write-error "No firewall log found.  Check firewall state" -ea continue
        }
```