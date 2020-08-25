layout: post
title: WinRM with IPv6 Kind of Disabled
featured-img: Powershell
comments: true
categories: [PowerShell]

## Winrm fails with a weird error when trying to connect from a machine with ipv6 on to a machine with it disabled.  test this and find out

```
Export-DfsrClone : Could not export the database clone for the volume D: to "F:\DfsrClone". Confirm that you are
running in an elevated Windows PowerShell session, the DFSR service is running, and that you are a member of the local
Administrators group. Error code: 0x80338113. Details: The WinRM client sent a request to an HTTP server and got a
response saying the requested HTTP URL was not available. This is usually returned by a HTTP server that does not
support the WS-Management protocol.
```

```
Update-DfsrConfigurationFromAD : Could not update the Active Directory Domain Service configuration on one of the
computers named: PDDPINF01PRIN. Confirm that you are running in an elevated Windows PowerShell session and are a
member of the local Administrators group on the destination computer. The destination computer must also be accessible
over the network, and have the DFSR service running. This cmdlet does not support WMI calls for the following or
earlier operating systems: Windows Server 2012. Details: The WinRM client sent a request to an HTTP server and got a
response saying the requested HTTP URL was not available. This is usually returned by a HTTP server that does not
support the WS-Management protocol.
```

above error from a machine with ipv6 enabled.  setting it to prever ipv4 over ipv6 fixes it.

question: where is it talking to?

https://support.microsoft.com/en-us/help/929852/guidance-for-configuring-ipv6-in-windows-for-advanced-users

Location:         HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\Tcpip6\Parameters\
Name:             DisabledComponents
Type:             REG_DWORD
Min Value:        0x00
Max Value:        0xFF (IPv6 disabled)

values:

0x00 - ipv6 enabled
0x01 - ipv6 disabled on tunnel interfaces
0x10 - ipv6 disabled on nontunnel interfaces
0x11 - ipv6 disabled on all interfaces (except loopback)
0x20 - prefer ipv4 over ipv6
0xff - ipv6 disabled



test ipv6 on against the flavors of ipv6 off

is it a mismatch that doesn't work?



