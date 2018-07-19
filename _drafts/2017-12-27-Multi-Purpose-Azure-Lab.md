---
layout: post
title: Multi-Purpose Azure Lab
categories: [PowerShell, Azure]
---
# Lab Lab Lab!

Ah Christmas, what a magical time of year to be in the office.  Nope, not ironic, it's really the best.  While everyone else is spending their vacation time to get away from the work, ringing phones, bad traffic, etc., I just get it all for free when everyone leaves.  There's nothing quite like a quick commute and a nice quiet office to put me in the holiday spirit.  Especially the last week of the year, that's got to be the filet mignon of work weeks.  Of course it's no good to just sit around surfing the youtubes, this magical week is best when you have a nice juicy project to occupy yourself with.  Planning that free time project can be more fun than planning a vacation.  And you get paid for it!  So what's it going to be this week?  Well how about something Azure?  And to really get a bang for the buck let's do a few things at once.  Hmm, I've got this new laptop with Hyper-V setup, maybe I could build a few VMs, a "domain", and run through the gamut of azure hybrid-cloud scenarios.  Let's see, we can start with a nice [Site to Site VPN Gateway](https://docs.microsoft.com/en-us/azure/vpn-gateway/vpn-gateway-howto-site-to-site-resource-manager-portal) to connect these VMs to Azure, then follow that with a serving of [Azure Site Recovery](https://docs.microsoft.com/en-us/azure/site-recovery/tutorial-migrate-on-premises-to-azure) to see about running those local VMs up in the cloud.  And if we still have room let's see about creating an [Azure AD tenant and syncing users](https://docs.microsoft.com/en-us/azure/active-directory/connect/active-directory-aadconnect) from our local "domain".  See?  It's totally the filet mignon of work weeks.  So let's get to it!

# Setup

The first thing to do is build our local resources.  We've got Hyper-V installed, so we need to make some VMs.  Nothing super special here, but we're going to need at a minimum these:

* __domain controller__ - can't have a domain without a domain controller. will also do DNS for the rest of the local VMs
* __dhcp__ - servers like dhcp, so let's do that and make it easy.
* __member server__ - not sure what we'll do with this one but it seems like we should have something non-specific that we can use later.  Plus it's a good way to test out domain membership and DHCP functionality
* __linux server__ - again, not sure what I'm going to do with it, but it'll probably come in handy. [Microsoft loves linux](https://blogs.technet.microsoft.com/windowsserver/2015/05/06/microsoft-loves-linux/) so I guess I should too.  Not domain joined but is on DHCP.
* __win10 member workstation__ - ?? why not?  another domain member
* __VPN Gateway__ - this will be the local endpoint of our Azure site to site connection.  In Hyper-V it should sit on the internal and external networks. Note:  This can't be server core, as the VPN client doesn't seem to work with core.  Put a GUI on this one.

So this layout should look something like this:

![Basic Lab Setup](/images/AzureLabDiagram.png)

I'll leave you, dear reader, up to your own devices for setting up the above and installing your operating systems.  I used trial ISOs from [here](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2016) but you do you and I'll catch you after the break.

---

Remember when every website was doing that "break" nonsense?  "Keep reading after the break".  What break?  you mean that line?  Yeah no, that's not a "break" or whatever, that's just a line, like a 2 pixel tall shaded area.  Anyway, where were we?  Oh yeah. Ok so you've got your servers installed and running along now so let's set them up.  

## Domain Controller

We'll start with the domain controller because everyone loves DNS, and a domain controller is the most fun way to get a DNS server (or something).  Cue the code!

```PowerShell
Add-WindowsFeature AD-Domain-Services -Restart -IncludeAllSubFeature -IncludeManagementTools
Install-ADForest -Name azurelab.local
```

Well that was uneventful.  I suppose you could get fancy with the domain setup if you want to spice things up, but the defaults should be fine for our purposes.

## DHCP Server

This should be slightly more interesting than the domain controller setup

```PowerShell
Add-Computer -domainname azurelab.local -credential administrator@azurelab.local
#wait for reboot
Add-WindowsFeature DHCP -Restart -IncludeAllSubFeature -IncludeManagementTools
#after reboot login with domain account (administrator@azurelab.local or another account if you made a second one like a responsible domain administrator)
Add-DhcpServerInDC -DnsName azurelab.local -IPAddress 192.168.1.2
Add-DhcpServerSecurityGroup -ComputerName dhcp
Add-DhcpServerDnsCredential -computername dhcp
Add-DhcpServerv4Scope -ComputerName dhcp -StartRange 192.168.1.10 -EndRange 192.168.1.250 -Name 'Internal Network' -State Active -SubnetMask 255.255.255.0
Set-DhcpServerv4OptionValue -Computername dhcp -DnsDomain azurelab.local -DnsServer 192.168.1.1
restart-service dhcpserver
```

> Note: we don't set the router option as we don't currently have any routing going on.  More on that later when we setup our vpn gateway.

Now we have DNS, DHCP and a domain.  All the comforts of home.  Just need to go through and setup our remaining servers and we'll have ourselves a nice little lab. At this point I'm going to 

# More Setup

Before we can setup our VPN connection we need to have something on the other end to connect to, so let's setup our Azure network.  Nothing too fancy, just like our local lab.  At this point we're just going to do a single vnet with a single subnet. You can do it from the portal if you want but here's the PowerShell commands and a magic button for the azure cloud shell.  Try it out! And since I always have to look up these commands here's the [URL](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-networks-create-vnet-arm-ps) with all the details.

```PowerShell
$resourcegroupname = 'AzurelabRG'
$location = 'SouthCentralUS'
New-AzureRMResourceGroup -name $resourcegroupname -location $location
$vnet1 = New-AzureRMVirtualNetwork -name vnet1 -resourcegroupname $resourcegroupname -location $location -AddressPrefix 10.0.0.0/16
Add-AzureRMVirtualNetworkSubnetConfig -name subnet1 -virtualnetwork $vnet1 -addressprefix 10.0.0.0/24
Set-AzureRMVirtualNetwork -VirtualNetwork $vnet1
```

[![Launch Cloud Shell](https://shell.azure.com/images/launchcloudshell.png "This is so cool!")](https://shell.azure.com/powershell) 

Now that we have a network in the cloud and a network in our laptop we just need to connect those two.  We're going to setup a VPN gateway in azure, and use certificate authentication to connect our local "gateway" VM.  We're going to follow the steps in this [article](https://docs.microsoft.com/en-us/azure/vpn-gateway/vpn-gateway-howto-point-to-site-resource-manager-portal) and see what happens.

After that things are going to get weird.  We're going to try to enable routing on our gateway vm, and see if we can share out that VPN connection to all the other VMs on our 'internal' network.  No idea if that works, but in theory it should.

Step 1, create a VPN Gateway for our new vnet via the portal.  There's probably a powershell way to do that but we'll figure that out later.  The portal says this could take a while so while that's churning along we'll switch over to our Hyper-V machine, and connect to our gateway VM.  From here we'll create and export our certificates.  You have a couple of options here.  You can install the AzureRM module on your vpn gateway machine and run all the commands on that machine, or you can run the cert commands on the vpngateway through invoke-command up to retreiving the certificate data, then run the azure stuff from the host. Either way should work.

```PowerShell
#create a root certificate
$rootcert = New-SelfSignedCertificate -Type Custom -KeySpec Signature `
-Subject "CN=P2SRootCert" -KeyExportPolicy Exportable `
-HashAlgorithm sha256 -KeyLength 2048 `
-CertStoreLocation "Cert:\CurrentUser\My" -KeyUsageProperty Sign -KeyUsage CertSign
#create a client certificate signed by the above root certificate
New-SelfSignedCertificate -Type Custom -KeySpec Signature `
-Subject "CN=P2SChildCert" -KeyExportPolicy Exportable `
-HashAlgorithm sha256 -KeyLength 2048 `
-CertStoreLocation "Cert:\LocalMachine\My" `
-Signer $rootcert -TextExtension @("2.5.29.37={text}1.3.6.1.5.5.7.3.2")
#convert the root certificate to upload to Azure
$Base64RootCert = [convert]::tobase64string($rootcert.RawData)
```

```PowerShell
#Login to Azure if you haven't already
#if you run this from a different machine than the gateway, you really only need the
#base64rootcert variable from the above code block
$Gateway = get-azurermvirtualnetworkgateway -resourcegroupname $ResourceGroupName
Set-AzureRmVirtualNetworkGateway -VirtualNetworkGateway $gateway -VpnClientAddressPool 172.16.201.0/24 -VpnClientProtocol SSTP,L2TP
Add-AzureRmVpnClientRootCertificate -VpnClientRootCertificateName P2SRootCert -VirtualNetworkGatewayName $gateway.Name -ResourceGroupName $resourcegroupname -PublicCertData $Base64RootCert
```

Ok, almost there.  So now we have a configured vpn gateway on the azure side, ready to take a connection from our local machine, complete with certificate authentication.  We just need to download and install the VPN client software.  Luckily we can do that from azure too, amazing!  Again, you can either setup the AzureRM module on your gateway machine, or run the below on your host, then copy it over manually.

```PowerShell
$clientURL = new-azurermvpnclientconfiguration -ResourceGroupName $resourcegroupname -Name $gateway.name -AuthenticationMethod EAPTLS | select-object -expandproperty vpnprofilesasurl
Invoke-WebRequest -Uri $clientURL -OutFile "$env:temp\VPNPackage.zip"

#after downloading the file, copy it over to the gateway machine
#the below command will need to be run from an administrative powershell session
Copy-VMFile -VM $(get-vm -name vpngateway) -SourcePath "$env:temp\VPNPackage.zip" -DestinationPath c:\temp\VPNPackage.zip -FileSource Host -Force -CreateFullPath
```

Now we need to go over to our gateway machine, extract the file, and install the VPN package

```powershell
#expand config file and create vpn connection
Expand-Archive -Path c:\temp\vpnpackage.zip -DestinationPath c:\temp\vpnpackage -force
C:\temp\vpnpackage\WindowsAmd64\VpnClientSetupAmd64.exe
```

And here we reach the end of our powershell commands, the rest of this we have to do through the GUI. (you did build the gateway machine with a GUI right?)
Select yes when prompted for installation, then you should be all set.  Click on network icon in the system tray and you should see something like this:

![VPN Client Installed](/images/AzureLabVPN.png)

Click on 'vnet1' to open the settings menu.  Select vnet1 again and click on the connect button to bring up the VPN connection menu. yep, click connect _again_ to connect to the vpn.

![VPN Client connection](/images/AzureLabVPNconnect.png)

A bunch of stuff should print out in that little status window thing and you should be connected.  If you click on the network menu in the systray again it should look like this now:

![VPN client connected](/images/AzureLabVPNConnected.png)

Success! Now if we look at the routing table we should see a route for our azure vnet (10.0.0.0/16) pointing to our vpn network connection (172.16.201.3)

![VPN client routing table](/images/AzureLabVPNRouteTable.png)

Ok that's enough for one day.  Next up we'll see about sharing this connection with the rest of the VMs on our hyper-v 'internal' network, and see what other trouble we can get into with that.