+++
author = "Bradley Bishop"
author_url = "https://github.com/bishopbm1"
categories = ["Bradley Bishop", "VMware", "pyVmomi", "Python", "Afinity Rule", "automation"]
date = "2019-01-28"
description = "Automated addition of VMware VM to Host Affinity Rules"
linktitle = ""
title = "Automated addition of VMware VM to Host Affinity Rules"
type = "post"

+++

# Background

VMware Affinity rules can solve many issues in a virtualized environment with out the need to create different clusters or datacenters to separate Virtual Machines. The specific need that we at Encore had for this was that when a vm is being cloned, if the VM was migrated to a different Host due to a vMotion, the custom spec with Kickstart files or Autounattended Syspreps would fail.

This was a random error that would only happened when vCenter or a host was overloaded. We didn't want to turn off the vMotion so we needed to find a different solution to keep our VMs on the same host during the automation task. We were able to do this using VM to Host Affinity rules and we were able to automate the process with the usage of pyVmomi.

We have published an action for creating affinity rules to an open source [StackStorm pack](https://github.com/StackStorm-Exchange/stackstorm-vsphere) to give the community an automated way to accomplish everything we will discuss below. You can go to the link to learn more about [StackStorm](https://stackstorm.com/).

Running an action to create Affinity Rules in StackStorm is as easy as:
```bash
st2 run vsphere.affinity_rule_create cluster_name="test-cluster" vm_names="test-vm1","test-vm2" host_names="test-host1","test-host2" rule_name="test-rule"
```

# Setup

You can install pyVmomi from pip into a virtualenv like the following:

```bash
virtualenv ~/vmware
source ~/vmware/bin/activate
pip install pyvmomi
```

# VMware Connection

pyVmomi makes it really easy to connect to the VMware vCenter environment to query information and perform all the necessary tasks needed.

Example of connecting to VMware vCenter unsecured:

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

# Creating Affinity Groups

The first thing that we have to do to create an affinity rule once we get connected to vCenter is to create two affinity groups. One for the VMware VMs and the second one is for the VMware Host machines. You can have many of each of these in the groups if you wish.

One important thing to keep in mind about this is that the VMs and the Hosts must already be created in VMware before you can add them to a group. And that Affinity rules and groups are applied at the cluster level so all the VMs and Hosts that you choose must live in the same cluster.

First we need to find the VMs and the Hosts that we will be adding to the Affinity Groups. Finding these objects can be done in the same manor. We can find many objects at the same time or we can find a single object, either way we have to return the objects as an array of objects so that they can be added to the group.

Once we have the objects we need to create separate groups that are created in a similar manor and add the objects to them.

### Finding Virtual Machines

To find VMs (either one or many) and add them to a VM groupyou can do the following:

```python
from pyVmomi import vim

vm_view = content.viewManager.CreateContainerView(client.rootFolder, [vim.VirtualMachine], True)

vm_names = ["test-vm1", "test-vm2"]
returned_vms = []
for vm_name in vm_names:
  for vm in vm_view.view:
      if vm.name == vm_name:
          returned_vms.append(vm)
          break

# Delete the View object once we are finished finding VMs
vm_view.Destroy()

# Decide on a name for the VM group and set it to a variable to be used later when creating the affinity rule
vm_group_name = "test-vm-group"

# Create VMware group
vm_group = vim.cluster.VmGroup(name=vm_group_name, vm=returned_vms)

# Create the group creation specification to be implemented at the end
vm_group_spec = vim.cluster.GroupSpec(info=vm_group, operation='add')
```

### Finding VMware Hosts

To find VMware Hosts (either one or many) and create a Host group is very similar to what we did for VMs:

```python
from pyVmomi import vim

host_view = content.viewManager.CreateContainerView(client.rootFolder, [vim.HostSystem], True)

host_names = ["test-host1", "test-host2"]
returned_hosts = []
for host_name in host_names:
  for host in host_view.view:
      if host.name == host_name:
          returned_hosts.append(host)
          break

# Delete the View object once we are finished finding Hosts
host_view.Destroy()

# Decide on a name for the Host group and set it to a variable to be used later when creating the affinity rule
host_group_name = "test-host-group"

# Create VMware group
host_group = vim.cluster.HostGroup(name=host_group_name, host=returned_hosts)

# Create the group creation specification to be implemented at the end
host_group_spec = vim.cluster.GroupSpec(info=host_group, operation='add')
```

# Creating Affinity Rule

Now that we have created both the VM group and the Host group above we can use that information to create the affinity rules. Doing this is acutally really similar to creating the VMware groups above. The only thing that you will need to have is the names of the groups that were created which we conviently saved to variables above.

To create Affinity rules:
```python
from pyVmomi import vim

# Create VMware VM to Host Rule
rule_obj = vim.cluster.VmHostRuleInfo(vmGroupName=vm_group_name,
                                     affineHostGroupName=host_group_name,
                                     name="test-rule",
                                     enabled=True,
                                     mandatory=True)

# Create the rule creation specification to be implemented at the end
rule_spec = vim.cluster.RuleSpec(info=rule_obj, operation='add')
```

# Implementing the Groups and Rules

Now that we have the groups and the rule information created we need to find the cluster to add these rules and groups to. All Affinity Rules and groups are assigned at the cluster level. So we need to find the cluster that these objects are associated with and add them to it. There are several ways to find the cluser that we are looking for. I will show you a few below.

Find by name:
```python
from pyVmomi import vim

cluster_view = content.viewManager.CreateContainerView(client.rootFolder, [vim.ClusterComputeResource], True)

custer_name = "test-cluster"
return_cluster = []
for cluster in cluster_view.view:
    if cluster.name == custer_name:
        return_cluster = cluster
        break

# Delete the View object once we are finished finding Hosts
cluster_view.Destroy()
```

Find by using one of the Hosts found above:
```python
first_host = returned_hosts[0]

return_cluster = first_host.parent
```

Find by using one of the VMs found above
```python
first_vm = returned_vms[0]

vm_host = first_vm.runtime.host

return_cluster = vm_host.parent
```

Regardless of the method that you use to get the cluster object. Once we have it we can finish up ad add all the information that we have created to the cluster so that VMware will create out Affinity Groups and Rule for us.

To add the information to the cluster:
```python
from pyVmomi import vim

# Create the Cluster Config Specification with the information that we created above
config_spec = vim.cluster.ConfigSpecEx(groupSpec=[vm_group_spec, host_group_spec],
                                      rulesSpec=[rule_spec])

# Add the information to VMware
add_task = cluster.ReconfigureEx(config_spec, modify=True)

# Check the status of the task until success
add_task.info.state
```

Now your new VMware Affinity Rule and Groups should be added to you VMware Cluster. This should help keep VMs on the correct host while you do any work that is needed.

-Bradley Bishop
