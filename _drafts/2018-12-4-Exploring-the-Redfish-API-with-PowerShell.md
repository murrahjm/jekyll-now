---
layout: post
title: Exploring the Redfish API with PowerShell
featured-img: DMTF_Redfish_logo
categories: [PowerShell]
---

Way back in the dark ages, when servers started coming on rails, and having more than one of everything, someone got the idea of adding something called a "baseboard management controller".  The idea there was that you could do out of band configuration of the hardware before there was any software on it.  Pretty cool idea, and we've been using RIBs, and ILOs, and DRACs ever since.  Log into the interface, poke around the bios settings, configure the disks, then boom, install your OS and you're off to the races.

Well I guess someone with stock in REST APIs decided that wasn't good enough.  Why login to some web interface with your fingers and eyeballs when you can do it with a script?  Add a task force, a few steering committees and shake vigorously.

Enter [Redfish](https://www.dmtf.org/standards/redfish), a common standard for hardware management via a REST API.  Now if you're like me, looking at the list of documentation on that site makes your head hurt, but have no fear!  I'm going to walk through a more concrete example with some actual hardware.

# Test Case

Imagine if you will a big brown box with an HPE logo arrives at your desk.  Shortly after that you get an email from you boss telling you about the brand new server that needs to get built.  Let's pretend it's going out to a remote field site to be a local file server or something.  So we'll pretend it needs Windows Server on it.  No VMs or anything, just a good 'ol Windows Server like mom used to make.  Oh and he says the other 150 or so are back in the loading dock.  So you could totally fire up that ILO, login, and do some configuration, but of course we know better than that!  Time to automate all these things and still have time for lunch.

## The Plan

We're going to dive into the weeds with the redfish api, at least the implementation that HP has on their hardware.  Though being a standard it should be pretty similar for Dell or IBM or [whomever](https://www.supermicro.com/solutions/Redfish.cfm).  For our specific imaginary scenario of building out a new server, we're going to look at the following tasks:

* query hardware information
* control power state
* configure bios settings
* configure raid arrays and logical drives
* configure ilo settings

And since this is a PowerShell blog we'll use PowerShell for most of our REST calls.  So let's do it!


> I'm using primarily Invoke-RestMethod to show the details of the REST API calls.  There may be more friendly vendor-specific PowerShell modules for these actions.  Check with your hardware vendor of choice if you want something a little more user friendly.

## Logging in and exploring the structure

To keep this part simple we're just going to use basic authentication.  There's a whole [session login thing](http://redfish.dmtf.org/schemas/DSP0266_1.0.html#session-management) but that seems complicated so for now we're going to keep it simple.

Step 1 of course is going to be plugging in the server and the ILO network interface.  We only need power though, we don't have to turn it on.  This part is a bit hardware specific, but an ILO with default settings will use DHCP to get an address and register a dns record in the format of `"ilo$serialnumber"`, i.e. `ilo2MAD64Q1`.  Other vendors are going to be who knows what so if you're following along at home do whatever you need to do to get your management interface accessible on the network.  If you can connect to it with your browser and login then you have what you need.

Ok!  so let's look at our first code sample:

```PowerShell
# add code to ignore the self-signed certificate.  This is for PS 5.1
# If using 6.0+ add the -SkipCertificateCheck to Invoke-RestMethod
add-type @"
    using System.Net;
    using System.Security.Cryptography.X509Certificates;
    public class TrustAllCertsPolicy : ICertificatePolicy {
        public bool CheckValidationResult(
            ServicePoint srvPoint, X509Certificate certificate,
            WebRequest request, int certificateProblem) {
            return true;
        }
    }
"@
[System.Net.ServicePointManager]::CertificatePolicy = New-Object TrustAllCertsPolicy
#set these values to your ilo/drac/whatever
$iloAddress = 'iloMXQ838068H.eprod.com'
$iloUserName = 'Administrator'
$iloPassword = '3VZTKF6Z'
#building the basic auth header info
$base64AuthInfo = [Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes(("{0}:{1}" -f $ilousername,$iloPassword)))
$header = @{
    Authorization=("Basic {0}" -f $base64AuthInfo)
    'Odata-version'='4.0'
    'Content-Type' ='application/json'
}

#perform a GET against the API
invoke-restmethod -uri "https://$iloaddress/redfish/v1/" -header $header
```

You should receive an output that looks similar to this:

```
@odata.context : /redfish/v1/$metadata#ServiceRoot.ServiceRoot
@odata.etag    : W/"2FCB4401"
@odata.id      : /redfish/v1/
@odata.type    : #ServiceRoot.v1_1_0.ServiceRoot
Id             : v1
AccountService : @{@odata.id=/redfish/v1/AccountService/}
Chassis        : @{@odata.id=/redfish/v1/Chassis/}
EventService   : @{@odata.id=/redfish/v1/EventService/}
JsonSchemas    : @{@odata.id=/redfish/v1/Schemas/}
Links          : @{Sessions=}
Managers       : @{@odata.id=/redfish/v1/Managers/}
Name           : HPE RESTful Root Service
Oem            : @{Hpe=}
RedfishVersion : 1.0.0
Registries     : @{@odata.id=/redfish/v1/Registries/}
SessionService : @{@odata.id=/redfish/v1/SessionService/}
Systems        : @{@odata.id=/redfish/v1/Systems/}
UUID           : b830a8f3-e1a1-583a-9031-94e6be9b9bfb
UpdateService  : @{@odata.id=/redfish/v1/UpdateService/}
```

This is the root endpoint for all redfish operations.  The first couple of objects are common to all endpoints, though the `@odata.id` and `@odata.type` are the most useful in this test scenario.  The majority of the rest of the output contains links to other areas.  For instance if we want to see info about the system itself we can use `/redfish/v1/Systems/`.  If we want to know about the "manager" device (that's the ilo/drac/whatever) we can use `/redfish/v1/Managers/`.  Notice how all the endpoints are plural (managerS,systemS)? Since this is an open standard, they can't make any assumptions about what kind or number of components you have.  Redfish doesn't judge you if you have two systems or three ILOs or whatever you want.

This will be important when we want to query these endpoints and subsequent components.  But first, I want to point out one other thing.  The `Oem` object above has a ton of data under it.  Let's make a small change to our above script and look at that output in more detail

```PowerShell
...
$output = invoke-restmethod -uri "https://$iloaddress/redfish/v1/" -Credential $ilocredential
$output.oem | convertto-json
```
```json
{
    "Hpe":  {
                "@odata.context":  "/redfish/v1/$metadata#HpeiLOServiceExt.HpeiLOServiceExt",
                "@odata.type":  "#HpeiLOServiceExt.v2_1_0.HpeiLOServiceExt",
                "Links":  {
                              "ResourceDirectory":  "@{@odata.id=/redfish/v1/ResourceDirectory/}"
                          },
                "Manager":  [
                                "@{DefaultLanguage=en; FQDN=ILOMXQ838068H.eprod.com; HostName=ILOMXQ838068H; Languages=System.Object[]; ManagerFirmwareVersion=1.35; ManagerType=iLO 5; Status=}"
                            ],
                "Moniker":  {
                                "ADVLIC":  "iLO Advanced",
                                "BMC":  "iLO",
                                "BSYS":  "BladeSystem",
                                "CLASS":  "Baseboard Management Controller",
                                "FEDGRP":  "DEFAULT",
                                "IPROV":  "Intelligent Provisioning",
                                "PRODABR":  "iLO",
                                "PRODFAM":  "Integrated Lights-Out",
                                "PRODGEN":  "iLO 5",
...
                            },
                "Sessions":  {
                                 "CertCommonName":  "ILOMXQ838068H.eprod.com",
                                 "CertificateLoginEnabled":  false,
                                 "KerberosEnabled":  false,
                                 "LDAPAuthLicenced":  true,
                                 "LDAPEnabled":  false,
                                 "LocalLoginEnabled":  true,
                                 "LoginFailureDelay":  0,
                                 "LoginHint":  "@{Hint=POST to /Sessions to login using the following JSON object:; HintPOSTData=}",
                                 "SecurityOverride":  false,
                                 "ServerName":  "PDLABAPP1PA"
                             },
                "System":  [
                               "@{Status=}"
                           ],
                "Time":  "2018-12-20T15:31:56Z"
            }
}
```

Holy Cow!  Well it's nothing if not verbose.  Since we're up at the root endpoint we're getting overview info about the system in general.  Similar to the `@odata` objects This `oem` structure is something we'll see throughout all our other endpoints.  As I understand it the idea is to put oem-specific info here that doesn't fit into the redfish general data structure.

Now that we have an idea of what we're looking at, let's browse around some more.  From here on I'll just be including code snippets of what changes from the first script we ran, so all the credential, and certificate stuff is just assumed at this point.  Let's look at the `Systems` endpoint:

```PowerShell
$output = invoke-restmethod -uri "https://$iloaddress/redfish/v1/Systems/1/" -header $Header
$output | ConvertTo-Json
```

```json
{
  "@odata.context": "/redfish/v1/$metadata#ComputerSystem.ComputerSystem",
  "@odata.etag": "W/\"4D3373ED\"",
  "@odata.id": "/redfish/v1/Systems/1/",
  "@odata.type": "#ComputerSystem.v1_4_0.ComputerSystem",
  "Id": "1",
  "Actions": {
    "#ComputerSystem.Reset": {
      "ResetType@Redfish.AllowableValues": [
        "On",
        "ForceOff",
        "ForceRestart",
        "Nmi",
        "PushPowerButton"
      ],
      "target": "/redfish/v1/Systems/1/Actions/ComputerSystem.Reset/"
    }
  },
  "AssetTag": "                                ",
  "Bios": {
    "@odata.id": "/redfish/v1/systems/1/bios/"
  },
  "BiosVersion": "U41 v1.42 (06/20/2018)",
  "Boot": {
    "BootSourceOverrideEnabled": "Disabled",
    "BootSourceOverrideMode": "UEFI",
    "BootSourceOverrideTarget": "None",
    "BootSourceOverrideTarget@Redfish.AllowableValues": [
      "None",
      "Cd",
      "Hdd",
      "Usb",
      "SDCard",
      "Utilities",
      "Diags",
      "BiosSetup",
      "Pxe",
      "UefiShell",
      "UefiHttp",
      "UefiTarget"
    ],
    "UefiTargetBootSourceOverride": "None",
    "UefiTargetBootSourceOverride@Redfish.AllowableValues": [
      "UsbClass(0xFFFF,0xFFFF,0xFF,0xFF,0xFF)",
      "PciRoot(0x2)/Pci(0x1,0x0)/Pci(0x0,0x0)/Pci(0x3,0x0)/Pci(0x0,0x0)/MAC(20677CD9EAC4,0x1)/IPv4(0.0.0.0)/Uri()",
      "PciRoot(0x2)/Pci(0x1,0x0)/Pci(0x0,0x0)/Pci(0x3,0x0)/Pci(0x0,0x0)/MAC(20677CD9EAC4,0x1)/IPv4(0.0.0.0)",
      "PciRoot(0x2)/Pci(0x1,0x0)/Pci(0x0,0x0)/Pci(0x3,0x0)/Pci(0x0,0x0)/MAC(20677CD9EAC4,0x1)/IPv6(0000:0000:0000:0000:0000:0000:0000:0000)/Uri()",
      "PciRoot(0x2)/Pci(0x1,0x0)/Pci(0x0,0x0)/Pci(0x3,0x0)/Pci(0x0,0x0)/MAC(20677CD9EAC4,0x1)/IPv6(0000:0000:0000:0000:0000:0000:0000:0000)",
      "IPv4(0.0.0.0)/Uri(http://mdt_ansible.eprod.com/MDT_ANSIBLE_UNATTENDED.iso)",
      "IPv6(0000:0000:0000:0000:0000:0000:0000:0000)/Uri(http://mdt_ansible.eprod.com/MDT_ANSIBLE_UNATTENDED.iso)",
      "PciRoot(0x2)/Pci(0x2,0x0)/Pci(0x0,0x0)/Scsi(0x0,0x4000)",
      "PciRoot(0x2)/Pci(0x2,0x0)/Pci(0x0,0x0)/Scsi(0x1,0x4000)"
    ]
  },
  "EthernetInterfaces": {
    "@odata.id": "/redfish/v1/Systems/1/EthernetInterfaces/"
  },
  "HostName": "PDLABAPP1PA",
  "IndicatorLED": "Blinking",
  "Links": {
    "ManagedBy": [
      {
        "@odata.id": "/redfish/v1/Managers/1/"
      }
    ],
    "Chassis": [
      {
        "@odata.id": "/redfish/v1/Chassis/1/"
      }
    ]
  },
  "LogServices": {
    "@odata.id": "/redfish/v1/Systems/1/LogServices/"
  },
  "Manufacturer": "HPE",
  "Memory": {
    "@odata.id": "/redfish/v1/Systems/1/Memory/"
  },
  "MemorySummary": {
    "Status": {
      "HealthRollup": "OK"
    },
    "TotalSystemMemoryGiB": 32
  },
  "Model": "ProLiant ML350 Gen10",
  "Name": "Computer System",
  "NetworkInterfaces": {
    "@odata.id": "/redfish/v1/Systems/1/NetworkInterfaces/"
  },
  "Oem": {
    "Hpe": {
      "@odata.context": "/redfish/v1/$metadata#HpeComputerSystemExt.HpeComputerSystemExt",
      "@odata.type": "#HpeComputerSystemExt.v2_5_0.HpeComputerSystemExt",
      "Actions": {
        "#HpeComputerSystemExt.PowerButton": {
          "PushType@Redfish.AllowableValues": [
            "Press",
            "PressAndHold"
          ],
          "target": "/redfish/v1/Systems/1/Actions/Oem/Hpe/HpeComputerSystemExt.PowerButton/"
        },
        "#HpeComputerSystemExt.SystemReset": {
          "ResetType@Redfish.AllowableValues": [
            "ColdBoot",
            "AuxCycle"
          ],
          "target": "/redfish/v1/Systems/1/Actions/Oem/Hpe/HpeComputerSystemExt.SystemReset/"
        }
      },
      "AggregateHealthStatus": {
        "AgentlessManagementService": "Unavailable",
        "BiosOrHardwareHealth": {
          "Status": {
            "Health": "OK"
          }
        },
        "Fans": {
          "Status": {
            "Health": "OK"
          }
        },
        "Memory": {
          "Status": {
            "Health": "OK"
          }
        },
        "PowerSupplies": {
          "PowerSuppliesMismatch": false,
          "Status": {
            "Health": "OK"
          }
        },
        "Processors": {
          "Status": {
            "Health": "OK"
          }
        },
        "SmartStorageBattery": {
          "Status": {
            "Health": "OK"
          }
        },
        "Storage": {
          "Status": {
            "Health": "OK"
          }
        },
        "Temperatures": {
          "Status": {
            "Health": "OK"
          }
        }
      },
      "Bios": {
        "Backup": {
          "Date": "06/20/2018",
          "Family": "U41",
          "VersionString": "U41 v1.42 (06/20/2018)"
        },
        "Current": {
          "Date": "06/20/2018",
          "Family": "U41",
          "VersionString": "U41 v1.42 (06/20/2018)"
        },
        "UefiClass": 2
      },
      "CurrentPowerOnTimeSeconds": 13475,
      "DeviceDiscoveryComplete": {
        "AMSDeviceDiscovery": "NoAMS",
        "DeviceDiscovery": "vMainDeviceDiscoveryComplete",
        "SmartArrayDiscovery": "Complete"
      },
      "EndOfPostDelaySeconds": null,
      "IntelligentProvisioningAlwaysOn": true,
      "IntelligentProvisioningIndex": 9,
      "IntelligentProvisioningLocation": "System Board",
      "IntelligentProvisioningVersion": "3.20.154",
      "IsColdBooting": false,
      "Links": {
        "HpeIpProvider": {
          "@odata.id": "/redfish/v1/systems/1/hpeip/"
        },
        "PCIDevices": {
          "@odata.id": "/redfish/v1/Systems/1/PCIDevices/"
        },
        "PCISlots": {
          "@odata.id": "/redfish/v1/Systems/1/PCISlots/"
        },
        "NetworkAdapters": {
          "@odata.id": "/redfish/v1/Systems/1/BaseNetworkAdapters/"
        },
        "SmartStorage": {
          "@odata.id": "/redfish/v1/Systems/1/SmartStorage/"
        },
        "USBPorts": {
          "@odata.id": "/redfish/v1/Systems/1/USBPorts/"
        },
        "USBDevices": {
          "@odata.id": "/redfish/v1/Systems/1/USBDevices/"
        },
        "EthernetInterfaces": {
          "@odata.id": "/redfish/v1/Systems/1/EthernetInterfaces/"
        }
      },
      "PCAPartNumber": "874585-001",
      "PCASerialNumber": "PWHVF0CRHB40AI",
      "PostDiscoveryCompleteTimeStamp": "2018-12-20T19:59:41Z",
      "PostDiscoveryMode": null,
      "PostMode": null,
      "PostState": "InPostDiscoveryComplete",
      "PowerAllocationLimit": 800,
      "PowerAutoOn": "Restore",
      "PowerOnDelay": "Minimum",
      "PowerOnMinutes": 1555,
      "PowerRegulatorMode": "Dynamic",
      "PowerRegulatorModesSupported": [
        "OSControl",
        "Dynamic",
        "Max",
        "Min"
      ],
      "SMBIOS": {
        "extref": "/smbios"
      },
      "ServerFQDN": "",
      "SmartStorageConfig": [
        {
          "@odata.id": "/redfish/v1/systems/1/smartstorageconfig/"
        },
        {
          "@odata.id": "/redfish/v1/systems/1/smartstorageconfig1/"
        }
      ],
      "VirtualProfile": "Inactive"
    }
  },
  "PowerState": "On",
  "ProcessorSummary": {
    "Count": 1,
    "Model": "Intel(R) Xeon(R) Gold 6134 CPU @ 3.20GHz",
    "Status": {
      "HealthRollup": "OK"
    }
  },
  "Processors": {
    "@odata.id": "/redfish/v1/Systems/1/Processors/"
  },
  "SKU": "877626-B21",
  "SecureBoot": {
    "@odata.id": "/redfish/v1/Systems/1/SecureBoot/"
  },
  "SerialNumber": "MXQ838068H",
  "Status": {
    "Health": "OK",
    "State": "Starting"
  },
  "Storage": {
    "@odata.id": "/redfish/v1/Systems/1/Storage/"
  },
  "SystemType": "Physical",
  "TrustedModules": [
    {
      "FirmwareVersion": "73.0",
      "InterfaceType": "TPM2_0",
      "Oem": {
        "Hpe": {
          "@odata.context": "/redfish/v1/$metadata#HpeTrustedModuleExt.HpeTrustedModuleExt",
          "@odata.type": "#HpeTrustedModuleExt.v2_0_0.HpeTrustedModuleExt",
          "VendorName": "STMicro"
        }
      },
      "Status": {
        "Health": "OK",
        "State": "Enabled"
      }
    }
  ],
  "UUID": "36373738-3632-584D-5138-333830363848"
}
```

Note that in my test scenario I only have a single server so I use `/Systems/1/` to reference the "first" system in my single-system ...system.



PUT a body to here to submit a change
/Systems/1/smartstorageconfig/settings/
doing a GET to that will show you pending changes
do a system reset, wait for POST, then if it worked whatever was in settings goes into /smartstorageconfig/

that same model for BIOS and ILO

system reset is through a system action.  these can be kind of hardware specific


setting a value results in a status message.  that message can be looked up in the api itself, something like this:

/registrystore/registries/en/smartstoragemessages.v2_0_0.smartstoragemessages/

