+++
author = "Greg Perry"
author_url = "https://github.com/gsperry2011"
categories = ["Greg Perry", "Encore", "VMware", "ESXi", "iSCSI", "Storage"]
date = "2020-03-05"
description = "Changing iSCSI LoginTimeout on VMware Hosts with PowerCLI"
featured = ""
featuredalt = ""
featuredpath = ""
linktitle = ""
title = "Changing iSCSI LoginTimeout with PowerCLI"
type = "post"

+++

In this blog post we will be explaining how we are changing the iSCSI LoginTimeout and the reasons why we are doing it in this method.

## Background

Recently we were going through a code upgrade on our storage arrays and one of the recommendations from Pure support is to increase the iSCSI LoginTimeout prior to upgrading the code. For the uninitiated, VMware defines the LoginTimeout as *"Time in seconds the initiator will wait for the login response to finish."*  [Source](https://docs.vmware.com/en/VMware-vSphere/6.7/com.vmware.vsphere.storage.doc/GUID-7FCA31F2-FA13-4BFD-8057-5A36DC3FBC14.html)

We currently leave this set to the default of 5 seconds. The primary reason for increasing this is so that you don't login-storm an array when a ESXi host attempts to login to the array. If the host fails it's first try (by default it would wait 5 seconds) it will retry login for the same target equal to the LoginRetryMax which has a default of 4.

## Digging Deeper

You may be surprised - as I was - that you cannot configure this LoginTimeout on a per-target basis. It can only be changed at the iSCSI adapter level so it will effect all targets connected to the adapter you change it on (confirmed w/ VMware support). 

When you rescan an entire cluster and/or VMware datacenter it rescans hosts sequentially (prevents login-storms) When you have a failed target during a rescan, it will take exactly as long as (LoginTimeout * LoginRetryMax) to fail each target. This can make storage rescans take a very long time.

## Hypothetical Situation:

- 5 VMware hosts
- 1 storage array with 10 LUNs on it
- All 5 hosts see all 10 LUNs. We experience an issue where we need to rescan storage on the hosts and it fails to login to the array for some reason.
- All VMware settings default *EXCEPT* LoginTimeout

1. With the LoginTimeout set to 60 seconds: each host will take 60 seconds to fail each target the first time. It will retry 4 times so each host takes 2400 seconds to rescan. (10 LUNs * 60 seconds each * 4 retries), Rescanning the entire cluster takes 12,000 seconds! 3 hours!! (2400 seconds per host * 5 hosts in total)
1. With the LoginTimeout set to 5 seconds: each host will take 5 seconds to fail each target the first time. It will retry 4 times so each host takes 200 seconds to rescan, entire cluster done in 1000 seconds or 16 minutes. 

Now imagine that we have a bigger environment with 100 hosts and 200 LUNs. We might not be able to do a storage rescan in under a day with a high LoginTimeout and failed targets in the mix and keep in mind you won't know if targets are going to fail prior to initiating the rescan!

## PowerCLI to Change LoginTimeout

This script will set the LoginTimeout on *ALL* iSCSI HBAs on *ALL* hosts in vCenter. This script is rather quick and in our testing we can change 15 hosts (roughly 20 iSCSI HBA's) in ~3 seconds.

It will only allow you to change the timeout between 5 - 60 seconds (the highest i've seen recommended)

Example command: `./Set-ScsiHBALoginTimeout.ps1 -vcenter 'vcenter.fqdn.tld' -user 'domain\user' -pass 'pass' -timeout_sec 10`

```powershell
param(
    $vcenter,
    $user,
    $pass,
    $timeout_sec
)

# Make sure we connect to vcenter
try{
	Connect-VIServer -Server $vcenter -user $user -pass $pass -force -ErrorAction Stop | out-null
   }
Catch{
	write-host "WARN: failed to connect to $vcenter, bailing"
	exit
}

# Verify the user enters a timeout between 5-60
if($timeout_sec -notin 5..60){
    write-host "ERROR: Please enter a timeout_sec between 5 and 60 seconds"
    exit
}

write-host "INFO: Starting process to change LoginTimeout"

$vmhs = get-vmhost | sort-object Name

foreach($h in $vmhs){
    $iscsi_hbas = $h | get-vmhosthba -type iscsi

    if($iscsi_hbas -eq $null){
	write-host "$h does not have an Iscsi adapter"
    }
    else{
	foreach($hba in $iscsi_hbas){
	    $esxcli = get-esxcli -vmhost $h
	    $esxcli.iscsi.adapter.param.set($hba.device,$false,'LoginTimeout',$timeout_sec) | out-null
	    write-host "INFO: Host: $h HBA:" $hba "changed LoginTimeout in $timeout_sec seconds."
	}
    }
}

write-host "INFO: Script completed"
```

Example output of script run:

![Example Run](/img/2020/03/script_example_run.png)

## Checking current LoginTimeout value

![GUI](/img/2020/03/logintimeout_gui.png)


## A Solution that Works for Our Customers

This environment has multiple different storage vendors in it and who knows what the future will bring. For this reason and the rescan duration increase we have decided to leave the LoginTimeout set to 5 seconds - So if we get into a situation where we need to expand a datastore, present new LUNs, decommission old LUNs, etc. we can do them swiftly and it does not require lowering the timeout to do these tasks quickly.

What we chose to do instead is to script changing this LoginTimeout on all the hosts so it's impossible to miss any hosts or fat finger the timeout value. It also allows us to change it on command when needed. Just prior to doing the Pure code upgrade we will increase the timeout to what they recommend (30 seconds), do the code upgrade, and then set the hosts back to the 5 second timeout.