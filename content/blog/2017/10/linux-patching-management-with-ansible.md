+++
author = "Brian Alcorn"
author_url = "https://github.com/balcornc"
categories = ["Brian Alcorn", "Encore"]
date = "2017-10-31"
description = "Centralized Linux Patching Activity Management for Consistency, Compliance, and Scalability"
featured = ""
featuredalt = ""
featuredpath = ""
linktitle = ""
title = "Linux Patching Management With Ansible - Phase One"
type = "post"

+++

# Background

While mulitple options exist for configuration and patching management, across multiple platforms, what we need is a vendor agnostic solution which, once built, will be easy to maintain, update, and enhance - and that won't require significant training for new associates to begin using.  For RHEL, we use Satellite to manage repositories, host collections, licensing, and some standard configs.  

For ensuring hosts have patches successfully applied inside a tight timeline, Satellite provides feedback not unlike the steering wheel of a '78 Chevrolet pick-up - you're in the ballpark, but never really 100% sure.  For consistency and compliance, we need something with better real time feedback.

In addition to "mushy" patch deployment feedback, Satellite is limited in the scope of automation it will accommodate.  We need a solution that can eventually  be turned loose to manage all patching activities on a schedule, without additional human input.  Ideally, once hosts, roles, playbooks and variables are fleshed out, it's simply a matter of defining and implementing a schedule to govern patch management.  Human interaction will only be necessary for adding/removing hosts, or when Ansible reports an error.



# Phase One

Phase one focuses on not only automating patching (you can accomplish that with a cron job that executes `yum update -y` on each server), but also on intelligent execution with pre- and post- patching steps, as well as error handling/reporting.

If you're new to Ansible, head over to https://www.ansible.com/get-started to learn how to install Ansible and become familiar with the basics.  


### Ansible Control Server Directory Structure

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
        |    |    |    |- <host_group> (optional)
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
        |        |- main.yaml
        |    
        |
        |- post_update
        |    |
        |    |- tasks
        |         |- main.yaml
        |
        |- yum_update
             |
             |- tasks
                  |- main.yaml

```


### Remote Host Directory Structure

The directory structure on remote hosts is as follows:

```text
/home/ansible_svc
    |
    |- env_verify.sh
    |- pre_update.sh
    |- pre_update.sh
    |- post_update.sh
```


### Ansible Control Server Configuration

The following examples provide a foundation for getting started with centralized Linux patching for Red Hat Enterprise Linux (RHEL) based systems. Refer to the directory structure wireframed above to aid in visualizing the end result.


#### /etc/ansible/inventory/group_vars/all/config
This configuration file defines the remote destination paths that files are written to by Ansible, and then used during playbook execution to run pre- and post-update actions.  Configurations stored in this file are universal to all remote hosts.  *This configuration file is mandatory.*

```text
# Destination paths for pre- and post-patching files
pre_update_script: /home/ansible_svc/pre_update.sh
post_update_script: /home/ansible_svc/pre_update.sh
pre_update_script: /home/ansible_svc/pre_update.sh
pre_update_script: /home/ansible_svc/pre_update.sh
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

```yaml
---
#------------------------------------------------------------------------------
# Description: Playbook to perform 'yum update' on selected hosts.
#        NOTE: This playbook will also reboot hosts if kernel is updated.
#      Author: Brian Alcorn
#        Date: 2017.10.24
#
#------------------------------------------------------------------------------

- name: Performing yum update on host(s)
  hosts: all
  become: yes
  any_errors_fatal: false

  roles:
  - pre_update
  - yum_update
  - post_update
```


#### /etc/ansible/roles/update_config/tasks/main.yaml
This file is where the playbook's tasks are executed from.  

```yaml
---
#--------------------------------------------------------------
# Description: Tasks for configuring pre- and post-
#              update actions for AFG Linux hosts
#      Author: Brian Alcorn
#        Date: 2017.10.11
#--------------------------------------------------------------

- name: Check required variables are defined
  fail:
    msg: 'variable {{ item }} is not defined'
  when: item is not defined
  with_items:
    - post_update_script
    - post_update_conf.env

- name: Verify ansible_svc directory exists
  file:
    path: /home/ansible_svc
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


#### /etc/ansible/roles/update_config/files/host/\<hostname\>/env_verify.sh
This optional script is triggered by Ansible to run as a part of the post_update process.  Include any checks for running services, or commands/test jobs that need to be run here.  This script in this location is specific to an individual host.  *Note the similarity to the env_verify.sh script in /etc/ansible/roles/update_config/files/group/\<host_group\>/env_verify.sh.*

**Ex.** */etc/ansible/roles/update_config/files/host/testbuild1.local/env_verify.sh*

```bash
#!/bin/bash
#------------------------------------------------------------
# Description: Post update verification script for <hostname>
#      Author: Brian Alcorn
#        Date: 2017.10.25
#------------------------------------------------------------

errorCheck=0

log_info "=================================================="
log_info " Environment-specific check"
log_info "=================================================="

# Check to see if Apache is running
if service httpd status; then
    log_info "Apache is running"
else
    log_error "Apache is NOT running"
    errorCheck=1
fi

# Check to see if MySQL is running
if service mysqld status; then
    log_info "MySQL is running"
else
    log_error " MySQL is NOT running"
    errorCheck=1
fi

return $errorCheck
```


#### /etc/ansible/roles/update_config/files/host/\<hostname\>/pre_update.sh
This optional script is triggered by Ansible to run as a part of the pre_update process.  Include any services that need to be stopped, any pre-patch information output, etc.  This script in this location is specific to an individual host. *Note the similarity to /etc/ansible/roles/update_config/files/group/host_group/pre_update.sh*. If pre_update.sh exists in both the host and group locations, host will take precedence.

**Ex.** */etc/ansible/roles/update_config/files/host/testbuild1.local/pre_udpate.sh

```bash
#!/bin/bash
#------------------------------------------------------------
# Description: Pre update script for <hostname>
#      Author: Brian Alcorn
#        Date: 2017.10.25
#------------------------------------------------------------

/etc/init.d/httpd stop
/etc/init.d/mysqld stop

```


#### /etc/ansible/roles/update_config/files/group/\<host_group\>/env_verify.sh
This optional script is triggered by Ansible to run as a part of the post_update process.  Include any checks for running services, or  commands/test jobs that need to be run here.  This script in this location will apply to all hosts within the host_group.  *Note the similarity to the env_verify.sh script in /etc/ansible/roles/update_config/files/host/\<hostname\>/env_verify.sh.*

**Ex.** */etc/ansible/roles/update_config/files/group/test/env_verify.sh*

```bash
#!/bin/bash
#------------------------------------------------------------
# Description: Post update verification script for <hostname>
#      Author: Brian Alcorn
#        Date: 2017.10.25
#------------------------------------------------------------

errorCheck=0

log_info "=================================================="
log_info " Environment-specific check"
log_info "=================================================="

# Check to see if Apache is running
if service httpd status; then
    log_info "Apache is running"
else
    log_error "Apache is NOT running"
    errorCheck=1
fi

# Check to see if MySQL is running
if service mysqld status; then
    log_info "MySQL is running"
else
    log_error " MySQL is NOT running"
    errorCheck=1
fi

return $errorCheck
```


#### /etc/ansible/roles/update_config/files/group/\<host_group\>/pre_update.sh
This optional script is triggered by Ansible to run as a part of the pre_update process.  Include any services that need to be stopped, any pre-patch information output, etc.  This script in this location will apply to all hosts witin the host_group. *Note the similarity to /etc/ansible/roles/update_config/files/host/\<host_group\>/pre_update.sh*.  If pre_update.sh exists in both the host and group locations, host will take precedence.

**Ex.** */etc/ansible/roles/update_config/files/group/test/pre_udpate.sh

```bash
#!/bin/bash
#------------------------------------------------------------
# Description: Pre update script for <hostname>
#      Author: Brian Alcorn
#        Date: 2017.10.25
#------------------------------------------------------------

/etc/init.d/httpd stop
/etc/init.d/mysqld stop

```


#### /etc/ansible/roles/pre_update/tasks/main.yaml
The pre_update role checks to see if a pre_update.sh script exists on the remote in the selected location, and if it does, executes it.
```yaml
---
#------------------------------------------------------------
# Description: Check for existenace of pre_update.sh script and run it if
#              found.
#      Author: Rick Paxton
#        Date: 2017.03.26
#------------------------------------------------------------
- name: Checking if pre_update.sh script exists
  stat:
    path: /home/ansible_svc/pre_update.sh
  register: update_scripts

- name: Running pre update script
  command: sh /home/ansible_svc/pre_update.sh
  when: update_scripts.stat.exists == true
  ignore_errors: yes

```


#### /etc/ansible/roles/post_update/tasks/main.yaml
```yaml
---
#------------------------------------------------------------------------------
# Description: Check for existenace of post_update.sh script and if reboot
#              flag has been set; if so, reboot host and wait for restart.
#      Author: Rick Paxton
#        Date: 2017.03.26
#------------------------------------------------------------------------------
- name: Checking if post_update.sh script exists
  stat:
    path: /home/ansible_svc/post_update.sh
  register: update_scripts

- name: Checking if reboot flag exists
  stat:
    path: /tmp/reboot
  register: reboot

- name: Running post update script
  command: sh /home/ansible_svc/post_update.sh
  when:
    - update_scripts.stat.exists == true
    - reboot.stat.exists == false

- name: Clearing reboot flag
  file:
    path: /tmp/reboot
    state: absent
  when: reboot.stat.exists == true

- name: Rebooting host(s).
  shell: sleep 2 && shutdown -r now "Reboot required for updated kernel."
  async: 20
  poll: 0
  when: reboot.stat.exists == true

- name: Waiting for host(s) to reboot
  local_action: wait_for
  args:
    host: "{{ ansible_fqdn }}"
    port: 22
    delay: 20
    state: started
    timeout: 180
  when: reboot.stat.exists == true
```


#### /etc/ansible/roles/yum_update/tasks/main.yaml
This is the main task file for the yum_update role.  Hosts are updated via yum, and the newest installed kernel version is compared to the running kernel.  If the running kernel is older that the newest installed kernel, `/tmp/reboot` is created, and will be recognized by the post_update role which will trigger a reboot of the host.


```yaml
---
#------------------------------------------------------------------------------
# Description: Perform yum update on selected hosts and compare running
#              kernel version with last updated kernel version.
#      Author: Rick Paxton
#        Date: 2017.03.26
#       Notes: shell module used to compare last kernel and current
#------------------------------------------------------------------------------

- name: Updating all packages
  yum:
    name: '*'
    state: latest
  tags:
    - skip_ansible_lint

- name: Comparing last kernel and running kernel
  shell: |
    LAST_KERNEL=$(rpm -q --last kernel | perl -pe 's/^kernel-(\S+).*/$1/' | head -1)
    CURRENT_KERNEL=$(uname -r)

    if [[ $LAST_KERNEL != $CURRENT_KERNEL ]]; then
      # Set reboot flag
      touch /tmp/reboot
      # Shutdown/stop any services before reboot if exists.
      if [[ -f /home/ansible_svc/pre_reboot.sh ]]; then
        /home/ansible_svc/pre_reboot.sh
      fi
    fi
  tags:
    - skip_ansible_lint
```


### Pushing/Updating Remote Host Configuration

Once all relevant group_vars/host_vars files are created, all that is required to push configurations to remote hosts is to navigate to /etc/ansible/playbooks and execute `ansible-playbook update_config.yaml`.  If variable assignments exist in both group_vars and host_vars, the assignments in `host_vars` will take precedence.  Once this has been executed, and the results validated, we are ready to apply patches.  If any errors are encountered,   they will be reported in /var/log/ansible.log as well as displayed on-screen when manually executed from a command terminal.



### Patching Remote Hosts

Once the Ansible Control Node is fully configured, and all necessary configurations have been pushed to remote hosts, everything is in place for executing patching actions.  All that is required is to navigate to /etc/ansible/playbooks and execute `ansible-playbook update.yaml`.  The playbook will manage all the rest of the actions, including triggering existing pre_update.sh scripts, performing yum updates, rebooting hosts when the Linux kernel is udpated, and triggering post_update.sh scripts.  If any errors are encountered, they will be reported in /var/log/ansible.log as well as displayed on-screen when manually executed from a command terminal.



### Conclusion

By implementing the principles and approach described here, we have been able to reduce patching time in some of our environments from 90-120 minutes down to 20-30 minutes.  The number of commands entered by a human, if manually kicking off patching, has been reduced from a dozen or so commands for each server down to 1: `ansible-playbook update.yaml`.  Adding new hosts into the patching schedule requires little more than updating /etc/ansible/inventory/hosts to include the new hosts' FQDNs, and adding any new group_vars or host_vars configuration files that are desired.  Further customization is easily achievable with this framework, and addresses many of our scalability needs.
