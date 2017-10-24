+++
author = "Brian Alcorn"
author_url = "https://github.com/balcornc"
categories = ["Brian Alcorn", "Encore"]
date = "2017-10-20"
description = "Centralized Linux Patching Activity Management for Consistency, Compliance, and Scalability"
featured = ""
featuredalt = ""
featuredpath = ""
linktitle = ""
title = "Linux Patching Management With Ansible - Phase One"
type = "post"

+++

# Background

While mulitple options exist for configuration and patching management, across multiple platforms, what need a vendor agnostic solution which, once built, will be easy to maintain, update, and enhance - and that won't require significant training for new associates to begin using.  For RHEL, we use Satellite to manage repositories, host collections, licensing, and some standard configs.  

For ensuring hosts have patches successfully applied inside a tight timeline, Satellite provides feedback not unlike the steering wheel of a '78 Chevrolet pick-up - you're in the ballpark, but never really 100% sure.  For consistency and compliance, we need something with better real time feedback.

In addition to "mushy" patch deployment feedback, Satellite is limited in the scope of automation it will accommodate.  We need a solution that can eventually  be turned loose to manage all patching activities on a schedule, without additional human input.  Ideally, once hosts, roles, playbooks and variables are fleshed out, it's simply a matter of defining and implementing a schedule to govern patch management.  Human interaction will only be necessary for adding/removing hosts, or when Ansible reports an error.

Finally, our solution will need to be able to apply patches to environments that are connected with differing Satellite servers.

# Enter Phase One

Phase one focuses on not only automating patching (you can accomplish that with a cron job that executes `yum update -y` on each server), but also on intelligent execution with pre- and post- patching steps, as well as error handling/reporting.

If you're new to Ansible, head over to https://www.ansible.com/get-started to learn how to install Ansible and become familiar with the basics.  


### Ansible Control Server

The directory structure we've elected to leverage on the Ansible control node is as follows:

```text
/etc/ansible
    |
    |- inventory
    |   |
    |   |- group_vars
    |   |    |
    |   |    |- all
    |   |    |   |
    |   |    |   |- config
    |   |    |
    |   |    |- <host_group> (optional)
    |   |    |   |
    |   |    |   |- config
    |   |
    |   |- hosts
    |   |   
    |   |- host_vars
    |        |
    |        |- <hostname> (optional)
    |        |   |
    |        |   |- config
    |
    |- playbooks
    |   |
    |   |- update_config.yaml
    |   |- update.yaml
    |
    |
    |- roles
        |
        |- update_config
        |    |
        |    |- files
        |    |    |
        |    |    |- host
        |    |    |    |- <hostname> (optional)
        |    |    |            |- env_verify.sh
        |    |    |            |- pre_update.sh
        |    |    |
        |    |    |- group
        |    |    |    |- <groupname> (optional)
        |    |    |            |- env_verify.sh
        |    |    |            |- pre_update.sh
        |    |    |
        |    |    |- post_update.sh
        |    |    |- post_update_conf.env
        |    |
        |    |- tasks
        |         |- main.yaml
        |
        |- pre_update
        |    |
        |    |- tasks
        |    |    |- main.yaml
        |    |
        |    |- vars
        |         |- main.yaml
        |
        |- post_update
        |    |
        |    |- tasks
        |         |- main.yaml
        |
        |- yum_update
             |
             |- handlers
             |    |- main.yaml
             |
             |- tasks
             |    |- main.yaml
             |
             |- vars
                  |- main.yaml


```

#### /etc/ansible/inventory/group_vars/all/config
This configuration file defines the remote destination paths that files are written to by Ansible, and then used during playbook execution to run pre- and post-update actions.  Configurations stored in this file are universal to all remote hosts.  *This configuration     file is mandatory.*

```yaml
# Destination paths for pre- and post-patching files
pre_update_script: /opt/encore/home/ansible_svc/pre_update.sh
post_update_script: /opt/encore/home/ansible_svc/pre_update.sh
pre_update_script: /opt/encore/home/ansible_svc/pre_update.sh
pre_update_script: /opt/encore/home/ansible_svc/pre_update.sh
```

#### /etc/ansible/inventory/group_vars/\<host_group\>/config
This configuration file defines the local source paths for files whose contents are written by Ansible to remote hosts, and then used during playbook execution to run pre- and post-update actions.  Configurations stored in this file are specifc to a host_group.  *This    configuration file is optional.*

**Ex.** */etc/ansible/inventory/group_vars/test_dev/config*
```yaml
env_verify_src: "{{ role_path }}/files/group/test_dev/env_verify.sh"
pre_update_src: "{{ role_path }}/files/group/test_dev/pre_update.sh"
```
#### /etc/ansible/inventory/hosts
This file contains all the hosts that the Ansible control node will manage.
```text
[test_dev] #host_group
testbuild1.local  #host_record

```

#### /etc/ansible/inventory/host_vars/\<hostname\>/config
This configuration file defines the local source paths for files whose contents are written by Ansible to remote hosts, and then used during playbook execution to run pre- and post-update actions.  Configurations stored in this file are specific to a host.  *This         configuration file is optional.*

**Ex.** */etc/ansible/inventory/host_vars/testbuild1.local/config*
```yaml
env_verify_src: "{{ role_path }}/files/host/testbuild1.local/env_verify.sh"
pre_update_src: "{{ role_path }}/files/host/testbuild1.local/pre_update.sh"
```

#### /etc/ansible/playbooks/update_config.yaml
This is the playbook YAML file that refers to the `inventory` and `role` files defined here to manage the update files for remote hosts.

```yaml
---
#---------------------------------------------------------
# Description: Playbook for managing pre- and post-update
#              action configurations for remote hosts
#      Author: Brian Alcorn
#        Date: 2017.10.11
#---------------------------------------------------------

- name: Configure pre- and post-update action configurations
  hosts: all
  become: yes

  roles:
    - update_config
```

#### /etc/ansible/playbooks/update.yaml
This playbook YAML file refers to the `inventory` and `role` files to execute patching for remote hosts.


#### /etc/ansible/roles/update_config/tasks/main.yaml
This file is where the playbook's tasks are executed from.  

```yaml
---
#--------------------------------------------------------------
# Description: Tasks for configuring pre- and post-
#              update actions for AFG Linux hosts
#      Author: B. Alcorn
#        Date: 2017.10.11
#--------------------------------------------------------------

- name: Check required variables are defined
  fail:
    msg: 'variable {{ item }} is not defined'
  when: item is not defined
  with_items:
    - post_update_script
    - post_update_conf.env

- name: Verify svc_encore directory exists
  file:
    path: /opt/encore/home/ansible_svc
    state: directory
    mode: '0755'

- name: Copy universal post-update script and config, optional files to hosts

  copy:
    src: "{{ item.src }}"
    dest: "{{ item.dest }}"
    owner: "{{ user_id }}"
    group: "{{ user_id }}"
    mode: "{{ item.file_mode }}"
    backup: yes

  with_items:
    - { src: post_update.sh,
        dest: "{{ post_update_script }}",
        file_mode: '0755' }
    - { src: post_update_conf.env,
        dest: "{{ post_update_conf }}",
        file_mode: '0655' }

- local_action: stat path="{{ pre_update_src }}"
  register: pre_update_file
    
- copy: 
    src: "{{ pre_update_src }}"
    dest: "{{ pre_update_script }}"
    owner: "{{ user_id }}"
    group: "{{ user_id }}"
    mode: '0755'
    backup: yes    
  when: pre_update_file.stat.exists
 
- local_action: stat path="{{ env_verify_src }}"
  register: env_verify_file
 
- copy:
    src: "{{ env_verify_src }}"
    dest: "{{ env_verify_dest }}"
    owner: "{{ user_id }}"
    group: "{{ user_id }}"
    mode: '0755'
    backup: yes    
  when: env_verify_file.stat.exists

```


### Remote Hosts

The directory structure on remote hosts is as follows: 

```text
/opt/home/encore/ansibl_svc
    |
    |- env_verify.sh
    |- pre_update.sh
    |- pre_update.sh
    |- post_update.sh
```
