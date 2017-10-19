+++
author = "Bradley Bishop"
author_url = "https://github.com/bishopbm1"
categories = ["Bradley Bishop", "vmWare", "pyVmomi", "Disks", "Controllers", "automation"]
date = "2017-10-19"
description = "Automated addition of vmWare Disks and Controllers"
featured = ""
featuredalt = ""
featuredpath = "date"
linktitle = ""
title = "Automated addition of vmWare Disks and Controllers"
type = "post"

+++

When adding disks to vmWare VM using Python we need to use the vmWare SOAP API. To make this easier we use the pyVmomi module which can be downloaded here: [pyVmomi](https://github.com/vmware/pyvmomi) and this gives us alot of very helpful features that take out a lot of the guess work in adding new disks to a vmWare VM.

# vmWare Connection

pyVmomi makes it really easy to connect to the vmWare environment to query information and perform all the necessary tasks needed.

Example of connecting to vmWare unsecured:

```python
from pyVim.connect import SmartConnect, Disconnect
import requests
import ssl

requests.packages.urllib3.disable_warnings()
context = ssl._create_unverified_context()

si = SmartConnect(host="test.dev",
                  port=443,
                  user="username",
                  pwd="password",
                  sslContext=context)

client = si.RetrieveContent()
```

# Virtual Machine

The client can then be used to perform many different tasks, such as querying for a VM. Once you retrieve a VM object you can get all the information about the VM as well as make changes to the VM. There are many different ways to query for a VM but since the VM names can be duplicated we will use the uuid.

Example of querying  for a vmWare VM using the client from above:

```python
vm = client.searchIndex.FindByUuid(uuid="0000-1111-2222", vmSearch=True)
```

# vmWare Disks

One nice this is auto generating a disk configuration specification so that all the required objects are available to be given values. Where possible it also generates base values. They also give you the ability to generate other information that is needed.

## Blank Disk

You can generate basic needs to a vmWare Disk by:

```python
from pyVmomi import vim

disk = vim.vm.device.VirtualDisk()
disk.backing = vim.vm.device.VirtualDisk.FlatVer2BackingInfo()

output:
  (vim.vm.device.VirtualDisk) {
     dynamicType = <unset>,
     dynamicProperty = (vmodl.DynamicProperty) [],
     key = 0,
     deviceInfo = <unset>,
     backing = (vim.vm.device.VirtualDisk.FlatVer2BackingInfo) {
        dynamicType = <unset>,
        dynamicProperty = (vmodl.DynamicProperty) [],
        fileName = '',
        datastore = <unset>,
        backingObjectId = <unset>,
        diskMode = '',
        split = <unset>,
        writeThrough = <unset>,
        thinProvisioned = <unset>,
        eagerlyScrub = <unset>,
        uuid = <unset>,
        contentId = <unset>,
        changeId = <unset>,
        parent = <unset>,
        deltaDiskFormat = <unset>,
        digestEnabled = <unset>,
        deltaGrainSize = <unset>,
        deltaDiskFormatVariant = <unset>,
        sharing = <unset>,
        keyId = <unset>
     },
     connectable = <unset>,
     slotInfo = <unset>,
     controllerKey = <unset>,
     unitNumber = <unset>,
     capacityInKB = 0L,
     capacityInBytes = <unset>,
     shares = <unset>,
     storageIOAllocation = <unset>,
     diskObjectId = <unset>,
     vFlashCacheConfigInfo = <unset>,
     iofilter = (str) [],
     vDiskId = <unset>
  }
```

This gives you all the available options that are needed to successfully create a new disk for a VM. Not all the information is required to be filled out, a lot of the information has the ability to be auto generated when the disk is added to a vmWare VM. Some of the options need to be whole objects as well.

## Example Disk

Here is an example of basic information needed to create a VMware Disk:

```python
from pyVmomi import vim

disk = vim.vm.device.VirtualDisk()
disk.backing = vim.vm.device.VirtualDisk.FlatVer2BackingInfo()
disk.backing.diskMode = "persistent"
disk.capacityInKB = 1024

# This needs to be the current controller on the VM.
disk.controllerKey = 1

# This must be a vmWare datastore object
disk.backing.datastore = vim.Datastore("datastore")

# This number is a unique number for the disk on the current controller
disk.unitNumber = 1

output:
  (vim.vm.device.VirtualDisk) {
     dynamicType = <unset>,
     dynamicProperty = (vmodl.DynamicProperty) [],
     key = 0,
     deviceInfo = <unset>,
     backing = (vim.vm.device.VirtualDisk.FlatVer2BackingInfo) {
        dynamicType = <unset>,
        dynamicProperty = (vmodl.DynamicProperty) [],
        fileName = '',
        datastore = 'vim.Datastore:datastore',
        backingObjectId = <unset>,
        diskMode = 'persistent',
        split = <unset>,
        writeThrough = <unset>,
        thinProvisioned = <unset>,
        eagerlyScrub = <unset>,
        uuid = <unset>,
        contentId = <unset>,
        changeId = <unset>,
        parent = <unset>,
        deltaDiskFormat = <unset>,
        digestEnabled = <unset>,
        deltaGrainSize = <unset>,
        deltaDiskFormatVariant = <unset>,
        sharing = <unset>,
        keyId = <unset>
     },
     connectable = <unset>,
     slotInfo = <unset>,
     controllerKey = 1,
     unitNumber = 1,
     capacityInKB = 1024,
     capacityInBytes = <unset>,
     shares = <unset>,
     storageIOAllocation = <unset>,
     diskObjectId = <unset>,
     vFlashCacheConfigInfo = <unset>,
     iofilter = (str) [],
     vDiskId = <unset>
  }
```

Disk datastore needs to be a vmWare datastore object to properly add. The controller key can be found by querying for the current controller and getting the key. There can only be a max of 15 disks per controller. The unitNumber on the disk is unique to the disk per controller. This goes 0 to 15 with 7 being reserved and unusable.

# vmWare Controller

We also have a need to add a new controller to a vmWare VM when the current controller is full. The process to add a controller is very similar to adding a disk thanks to pyVmomi.

## Blank Controller

Example of a blank controller:

```python
from pyVmomi import vim

controller = vim.vm.device.VirtualSCSIController()

output:
  (vim.vm.device.VirtualSCSIController) {
     dynamicType = <unset>,
     dynamicProperty = (vmodl.DynamicProperty) [],
     key = 0,
     deviceInfo = <unset>,
     backing = <unset>,
     connectable = <unset>,
     slotInfo = <unset>,
     controllerKey = <unset>,
     unitNumber = <unset>,
     busNumber = 0,
     device = (int) [],
     hotAddRemove = <unset>,
     sharedBus = <unset>,
     scsiCtlrUnitNumber = <unset>
  }
```

This is very similar to the Disks configuration in that most of the options do not need to be filled out. However, some do, something to keep in mind is that while the busNumber is initialized at 0 this is already taken by the first controller on the VM. This busNumber can be 0 to 3 since there can only be 4 controllers per VM.

## Example Controller

Here is an example of a controller that has been filled in to be added.

```python
from pyVmomi import vim

controller = vim.vm.device.VirtualSCSIController()
controller.sharedBus = 'noSharing'
controller.busNumber = 1

output:
  (vim.vm.device.VirtualSCSIController) {
     dynamicType = <unset>,
     dynamicProperty = (vmodl.DynamicProperty) [],
     key = 0,
     deviceInfo = <unset>,
     backing = <unset>,
     connectable = <unset>,
     slotInfo = <unset>,
     controllerKey = <unset>,
     unitNumber = <unset>,
     busNumber = 1,
     device = (int) [],
     hotAddRemove = <unset>,
     sharedBus = 'noSharing',
     scsiCtlrUnitNumber = <unset>
  }
```

Once you have your configuration for the addition that you want to make to the vmWare VM you need to generate a configuration specification to be passed into a reconfigure VM task to send to vmWare. This accepts a list of configurationss that can be done at the same time if you wish. Keep in mind that if you need to generate a new controller to add a disk then that will need to be done before so that you can get auto generated information unless you specify it.

# Reconfigure VM Task

Example of how to add a disk using the VM object from above:

```python
from pyVmomi import vim

# Generate basic Disk information
disk = vim.vm.device.VirtualDisk()
disk.backing = vim.vm.device.VirtualDisk.FlatVer2BackingInfo()
disk.backing.diskMode = "persistent"
disk.capacityInKB = 1024
disk.controllerKey = 1
disk.backing.datastore = vim.Datastore("datastore")
disk.unitNumber = 1

# Generate virtual device Specification
disk_spec = vim.vm.device.VirtualDeviceSpec()
disk_spec.fileOperation = "create"
disk_spec.operation = vim.vm.device.VirtualDeviceSpec.Operation.add

# Add disk to the device spec
disk_spec.device = disk

config_spec = vim.vm.ConfigSpec()
config_spec.deviceChange = [disk_spec]

vm.ReconfigVM_Task(config_spec)
```

Config Spec output:

```python
(vim.vm.ConfigSpec) {
   dynamicType = <unset>,
   dynamicProperty = (vmodl.DynamicProperty) [],
   changeVersion = <unset>,
   name = <unset>,
   version = <unset>,
   uuid = <unset>,
   instanceUuid = <unset>,
   npivNodeWorldWideName = (long) [],
   npivPortWorldWideName = (long) [],
   npivWorldWideNameType = <unset>,
   npivDesiredNodeWwns = <unset>,
   npivDesiredPortWwns = <unset>,
   npivTemporaryDisabled = <unset>,
   npivOnNonRdmDisks = <unset>,
   npivWorldWideNameOp = <unset>,
   locationId = <unset>,
   guestId = <unset>,
   alternateGuestName = <unset>,
   annotation = <unset>,
   files = <unset>,
   tools = <unset>,
   flags = <unset>,
   consolePreferences = <unset>,
   powerOpInfo = <unset>,
   numCPUs = <unset>,
   numCoresPerSocket = <unset>,
   memoryMB = <unset>,
   memoryHotAddEnabled = <unset>,
   cpuHotAddEnabled = <unset>,
   cpuHotRemoveEnabled = <unset>,
   virtualICH7MPresent = <unset>,
   virtualSMCPresent = <unset>,
   deviceChange = (vim.vm.device.VirtualDeviceSpec) [
      (vim.vm.device.VirtualDeviceSpec) {
         dynamicType = <unset>,
         dynamicProperty = (vmodl.DynamicProperty) [],
         operation = 'add',
         fileOperation = 'create',
         device = (vim.vm.device.VirtualDisk) {
            dynamicType = <unset>,
            dynamicProperty = (vmodl.DynamicProperty) [],
            key = 0,
            deviceInfo = <unset>,
            backing = (vim.vm.device.VirtualDisk.FlatVer2BackingInfo) {
               dynamicType = <unset>,
               dynamicProperty = (vmodl.DynamicProperty) [],
               fileName = '',
               datastore = 'vim.Datastore:datastore',
               backingObjectId = <unset>,
               diskMode = 'persistent',
               split = <unset>,
               writeThrough = <unset>,
               thinProvisioned = <unset>,
               eagerlyScrub = <unset>,
               uuid = <unset>,
               contentId = <unset>,
               changeId = <unset>,
               parent = <unset>,
               deltaDiskFormat = <unset>,
               digestEnabled = <unset>,
               deltaGrainSize = <unset>,
               deltaDiskFormatVariant = <unset>,
               sharing = <unset>,
               keyId = <unset>
            },
            connectable = <unset>,
            slotInfo = <unset>,
            controllerKey = 1,
            unitNumber = 1,
            capacityInKB = 1024,
            capacityInBytes = <unset>,
            shares = <unset>,
            storageIOAllocation = <unset>,
            diskObjectId = <unset>,
            vFlashCacheConfigInfo = <unset>,
            iofilter = (str) [],
            vDiskId = <unset>
         },
         profile = (vim.vm.ProfileSpec) [],
         backing = <unset>
      }
   ],
   cpuAllocation = <unset>,
   memoryAllocation = <unset>,
   latencySensitivity = <unset>,
   cpuAffinity = <unset>,
   memoryAffinity = <unset>,
   networkShaper = <unset>,
   cpuFeatureMask = (vim.vm.ConfigSpec.CpuIdInfoSpec) [],
   extraConfig = (vim.option.OptionValue) [],
   swapPlacement = <unset>,
   bootOptions = <unset>,
   vAppConfig = <unset>,
   ftInfo = <unset>,
   repConfig = <unset>,
   vAppConfigRemoved = <unset>,
   vAssertsEnabled = <unset>,
   changeTrackingEnabled = <unset>,
   firmware = <unset>,
   maxMksConnections = <unset>,
   guestAutoLockEnabled = <unset>,
   managedBy = <unset>,
   memoryReservationLockedToMax = <unset>,
   nestedHVEnabled = <unset>,
   vPMCEnabled = <unset>,
   scheduledHardwareUpgradeInfo = <unset>,
   vmProfile = (vim.vm.ProfileSpec) [],
   messageBusTunnelEnabled = <unset>,
   crypto = <unset>,
   migrateEncryption = <unset>
}
```

With this information, we can now effectively add many disks to a VM without worrying about running out of controller space since we can effectively add more controllers when needed.

-Bradley Bishop