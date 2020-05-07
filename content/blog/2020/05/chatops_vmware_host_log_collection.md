+++
author = "Greg Perry"
author_url = "https://github.com/gsperry2011"
categories = ["Greg Perry", "Encore", "VMware", "ESXi", "logs", "chatops", "Stackstorm", "Bolt", "puppet"]
date = "2020-05-07"
description = "How Encore is Utilizing Chatops for VMWare Support Cases"
featured = ""
featuredalt = ""
featuredpath = ""
linktitle = ""
title = "Utilizing Chatops for VMWare Support Cases"
type = "post"

+++

In this post we will be explaining how we have created a chatops workflow that enables us to upload VMware ESXi log bundles directly to a VMware support case from our chat client of choice. The code that makes all of this work is at the bottom of this article.

## Chatops Introduction

For the those who have never heard of chatops before; To me, it means the ability to call automation workflows from your favorite chat client. This means our automation team can code up a workflow and then connect it to a dedicated channel in slack where other engineers / admins can call these commands and pass parameters to them.

There are a few things I find very cool about Chatops:
- It allows administrators who don't understand code to leverage automation others have written
- It enables us to do certain tasks from wherever we might be, on whatever device we have available. Ever wish you could upload vmware logs from your cell phone after getting that email from support when you are out of the office?
- It can be a huge time saver! No need to get on your laptop so you can 2FA into an environment. You'd spend 10 minutes logging in to do a 5 minute job.

## Background

Anyone who has ran a VMware environment for some time knows that the first thing VMware support will ask when you open a case is to upload a log bundle to them. Depending on the nature of the case you might need to extract log bundles from *ALL* of the hosts, in other cases you might need to extract a log bundle from the same host multiple times after a specific event has occured.

To collect a log bundle usually means:
- Hopping into the jumpbox for the specific environment
- Logging into the host(s) via SSH or logging into the vCenter console to generate the bundle
- Uploading that bundle from your download location up to VMware support.

Not too challenging but definitely time consuming - and definitely something we can help automate!

## Overview of Chatops Action

From a chatbops perspective this is what the action looks like:

![help](/img/2020/05/slack_help_command.png)

Example of a run where I collect a log bundle from my-vmware-host.fqdn.tld and upload it to case #12345678901:

![help](/img/2020/05/logupload_full.png)

You might find it interesting that Dot Matrix (stackstorm) immediately queued my action when I called it, and it took about ~6 minutes for it to complete. You can see the timestamp of `05-06-2020@15:20` in the log bundle name, the time at which it was initiated (3:20PM). Stackstorm took less than a minute to get from slack to the vmware host!

## Chatops Tool Stack

To create this chatops action we are using a combination of tools, nearly the entire list is open source software and can be downloaded for free:

- Stackstorm - https://stackstorm.com/ 
  - Interpreting chatops commands entered in slack
  - Posting the results of the workflow to the caller via slack
  - Executing Bolt task
  - Populates environment variables for use in Bolt task
- Puppet Bolt - https://puppet.com/docs/bolt/latest/bolt.html
  - Running the actual code that connects to the vmware host, generates the bundle, uploads bundle to vmware support
- Slack - https://slack.com/
  - Slack is our chat platform of choice. It has very nice integrations available for Chatops.

## Deep Dive

If you're still reading by now I assume you are ready to see some code on how we put this together! To make things easier for you the general idea is: `shell script -> bolt task -> stackstorm action -> stackstorm workflow -> stackstorm alias`

The steps involved in this shell script are:

- Generate and download a log bundle via API call at `https://ESXHostnameOrIPAddress/cgi-bin/vm-support.cgi` Explained here: https://kb.vmware.com/s/article/1010705
- Initiate an SFTP connection to vmware support
- Create a directory that matches our support case number.
  - Unfortunately VMware does not give us `ls` command access on the sftp server so we can't check if this exists already, thankfully it's not show stopping if it does already exist.
- Upload our log bundle to said directory
- Clean up temporary bundle download location

###  Bolt task code

vmware_generate_and_upload_log_bundle.sh:

This is the shell script that does the majority of the work:

Note: All variables starting with `$PT_` are supplied via bolt.

```
#!/bin/bash
# Download log bundle from esx host API to the download directory and then upload it to
# VMWare support's SFTP site.

# Determine if a download directory was specified via bolt. Download directory is
# the location the log bundle is downloaded to on the system that runs this script. 
if [ -z ${PT_download_directory+x} ]
then
    download_directory=/tmp/
else
    download_directory="$PT_download_directory"
fi

# EX: DD-MM-YYYY@HH:MM
curr_datetime=$(date +"%d-%m-%Y@%H:%M")

# EX: my-vmware-host.fqdn.tld_DD-MM-YY@HH:MM.zip
filename="${PT_esx_host}_${curr_datetime}.tgz"

# EX: /tmp/my-vmware-host.fqdn.tld_DD-MM-YY@HH:MM.zip
filename_with_path="${download_directory}${filename}"

# Checking HTTP status of the host before we attempt to collect logs.
# We cannot curl vmhost/cgi-bin/vm-support.cgi or it will trigger a log collection and our actual log collection
# curl we have after this check will fail - you will get a 409 return code.
http_code=$(curl --insecure -u $PT_esx_username:$PT_esx_password -s -o /dev/null -I -w "%{http_code}" https://$PT_esx_host)

if [ "$http_code" == '200' ]
then
    # Download log bundle from esx host API to the download directory
    # EX: curl -s --insecure -u username:password https://vmhost.fqdn.tld/cgi-bin/vm-support.cgi -o /tmp/vmhost.fqdn.tld_DD-MM-YYYY@HH:MM.gzip
    curl -sS --insecure -u $PT_esx_username:$PT_esx_password https://$PT_esx_host/cgi-bin/vm-support.cgi -o $filename_with_path
else
    echo "ERROR: Got http status of ${http_code} from https://${PT_esx_host}/cgi-bin/vm-support.cgi - Bailing!"
    exit 1
fi

# The hostname of the system where the download will be stored
hostname=$(hostname)

# Verify we can locate the specified log file (sanity checking vmware)
if [ ! -f "$filename_with_path" ]
then
    echo "ERROR: Unable to locate file ${filename_with_path} on ${hostname} - Bailing!"
    exit 1
fi

# Checking vmware site
http_code=$(curl -s -o /dev/null -I -w "%{http_code}" ${PT_sftp_site})

# Verify the sftp site is up
if [ "$http_code" != '200' ]
then
    echo "ERROR: Got http status of ${http_code} from ${PT_sftp_site} - Bailing!"
    exit 1
fi

# All of the commands following <<EOF below will be executed within the sftp session.
# The put command within is where the log bundle gets uploaded to VMware's SFTP site.
# Note: the mkdir command might create an error if the directory already exists, but we cannot validate
# that it exists as we do not have access to the 'ls' command on the sftp server.

# Establish sftp connection and execute payload
sshpass -p $PT_sftp_password sftp -o 'StrictHostKeyChecking no' $PT_sftp_username@sftpsite.vmware.com <<EOF 

mkdir $PT_vmware_sr_number
cd $PT_vmware_sr_number
put $filename_with_path
exit

EOF

echo "${filename} successfully uploaded to VMware SR: ${PT_vmware_sr_number}."

# Clean up the extracted log bundle from the node
rm $filename_with_path
```

and the other piece of the bolt task:

vmware_generate_and_upload_log_bundle.json
```
{
  "description": "Generates a log bundle on an esxi host",
  "implementations": [
    {"name": "vmware_generate_and_upload_log_bundle.sh", "requirements": ["shell"]}
  ],
  "parameters": {
    "esx_username": {
      "description": "Username to login to the esx host. Generally root.",
      "type": "String[1]"
    },
    "esx_password": {
      "description": "Password to login to the esx host. Generally root password.",
      "type": "String[1]",
      "sensitive": true
    },
    "esx_host": {
      "description": "Esx host to collect log bundle from.",
      "type": "String[1]"
    },
    "download_directory": {
      "description": "Location in which the .zip is downloaded to from the esx host. Default /tmp/",
      "type": "Optional[String]"
    },
    "sftp_site": {
      "description": "URL to the SFTP site.",
      "type": "String[1]"
    },
    "sftp_username": {
      "description": "Username to login to the SFTP site.",
      "type": "String[1]"
    },
    "sftp_password": {
      "description": "Password to login to the SFTP site.",
      "type": "String[1]",
      "sensitive": true
    },
    "vmware_sr_number": {
      "description": "Must be 11 digit VMware SR #. Directory on the SFTP server the log will upload to.",
      "type": "String[1]"
    }
  }
}
```

### Stackstorm Code

Now that we have a functional Bolt task to do want we want, we can simply roll it into a Stackstorm workflow that can be called via slack.


This Stackstorm action calls the bolt task above: vmware_host_collect_and_upload_log_bundle.yaml

```
---
description: "Gather log bundle from specified ESX host and upload that log bundle to a VMware SR #"
enabled: true
runner_type: orquesta
entry_point: workflows/vmware_host_collect_and_upload_log_bundle.yaml
name: vmware_host_collect_and_upload_log_bundle
pack: encore
parameters:
  vmware_sr_number:
    type: string
    description: "11 digit VMware SR number"
    required: true
  esx_host:
    type: string
    description: "FQDN or IP of ESX host to collect logs from"
    required: true
  download_server:
    type: string
    description: "FQDN or IP of Linux system to temporarily place logs for upload to support"
    default: "{{ st2kv.system.stackstorm.server }}"
```




This Stackstorm workflow calls the Stackstorm action above: vmware_host_collect_and_upload_log_bundle.yaml

One of the key things here is that environment variables are being populated for the usernames and passwords being used. This means that they are NOT visible on the system running this code.
```
---
version: 1.0

description: Workflow to collect and upload a log bundle from a specified ESX host

input:
  - vmware_sr_number
  - esx_host
  - download_server

vars:
  - action_error: ''
  - action_result: ''

output:
  - action_error: "{{ ctx().action_error }}"
  - action_result: "{{ ctx().action_result }}"

tasks:
  plan_run:
    action: encore.bolt_plan
    input:
      server_fqdn: "{{ ctx().download_server }} "
      os_type: "linux"
      plan: "encore_rp::vmware_generate_and_upload_log_bundle"
      params:
        esx_host: "{{ ctx().esx_host }}"
        vmware_sr_number: "{{ ctx().vmware_sr_number }}"
      st2kv_vars:
        esx_username: "system.vmware.vmhost.username"
        esx_password: "system.vmware.vmhost.password"
        sftp_site: "system.vmware.sftp.site"
        sftp_username: "system.vmware.sftp.username"
        sftp_password: "system.vmware.sftp.password"
      bolt_retry_count: "1"
    next:
      - when: "{{ succeeded() }}"
        publish:
          - action_result: "{{ result().output.run[0].result.value }}"
        do:
          - noop
      - when: "{{ failed() }}"
        publish:
          - action_error: "{{ result().errors[0].result }}"
        do:
          - fail
```

The Stackstorm alias is how you configure what occurs in slack in reguards to the Stackstorm workflow above.
vmware_host_collect_and_upload_log_bundle.yaml:

```
---
name: vmware_host_collect_and_upload_log_bundle
pack: encore
description: Uploads specified VMware host log bundle to (11 digit) VMware ticket number.
action_ref: encore.vmware_host_collect_and_upload_log_bundle

formats:
  - "vmware host log upload {{ esx_host }} {{ vmware_sr_number }}"

result:
  format: |
    status: {{ execution.status }}
    output:
    -------------------------------
    {% if execution.status == 'succeeded' %}
      {{ execution.result.output.action_result['_output'] }}
    {% elif execution.status == 'failed' %}
      {{ execution.result.output.action_error }}
    {% endif %}
```


