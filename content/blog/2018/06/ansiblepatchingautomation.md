+++
author = "Brian Alcorn"
author_url = ""
categories = ["Brian Alcorn", "Encore", "Ansible", "Patch", "Patching", "DevOps"]
date = "2018-06-05"
description = "How we automated Red Hat Enterprise Linux OS patching to reduce time-to-production and human error, while improving compliance and risk management posture."
featuredpath = "date"
linktitle = ""
title = "Automating Red Hat Enterprise Linux Patching with Ansible (Part 1 of 2)"
type = "post"
+++

*In this first installment of a two-part series, we'll be going over Phase One, the build out of the "core" patching and reboot functionality on Ansible.* 

# History

Around a year ago, we began working with a customer whose Red Hat Enterprise Linux (RHEL) 6 and 7 OS patching process was being conducted manually.  This required highly skilled administrators focused solely on patching.  Documentation was eschewed in favor of tribal knowledge and manual command entry at the command line presented moderate to high risk during server patching.  Some of the involved environments included the handling of Personally Identifiable Information (PII) as well as data and information subject to Payment Card Industry (PCI) compliance requirements.

[PCI Security Standards Council](https://www.pcisecuritystandards.org/documents/Prioritized-Approach-for-PCI_DSS-v3_2.pdf) states:

>Ensure that all system components and software are protected from known vulnerabilities by installing applicable vendor-supplied security patches.  Install critical patches within one month of release.

Because of this and other PCI compliance requirements, all the customer's RHEL servers needed to be patched at least once monthly.  Patching needed to follow a *DEVELOPMENT -> STAGING -> PRODUCTION* workflow to minimize risks to the production environments. Because of the complexities involved in a fully manual patching process, meeting PCI compliance standards was a constant chase with inconsistent results.

Dissatisfied with the high costs, high complexity and consistent underperformance of the manual patching process, the customer sought ways to improve operational efficiency and reduce costs, expressing interest in leveraging automation where it made sense to do so.

# Enter Ansible

To minimize complexity, the solution was developed on [Ansible](https://www.ansible.com/), in conjunction with common Linux tools such as YUM, Bash scripts and SSH keys.  No agent software installation or additional security infrastructure was required, thus the customer incurred zero appreciable cost for the technology. 

Implementation was carried out in three phases:

1. Automate Basic Patching and Endpoint Reboots
2. Automate Standard Pre- and Post-patching Activities
3. Automate Application/Host-specific Pre- and Post-patching Activities

## Phase One - Automate Basic Patching and Endpoint Reboots

The first phase of the project focused on the installation and configuration of Ansible on a control node, as well as developing the core patch installation and reboot functionality.  Automation of patch installation and reboots, without any change to pre- and post-patching activity processes saw an immediate benefit of speedier patch installation as well as a marked reduction in human errors being introduced.

Before development of patching automation could begin in earnest, a foundational directory structure needed to be established for Ansible.  Following is a visual representation:

![Ansible Directory Structure](/img/2018/06/PatchingAutomationFileTree.png)

### Inventory

The `inventory` subdirectory is home to the `hosts` file and Ansible references to know what systems to work with.  For our customer, the hosts file contains server FQDNs (preferred) and IP addresses (where hostname usage is prohibitive) for all Ansible managed endpoints. For example *(these are NOT real hostnames)*:


*/ansible/invetory/hosts*
```
[group1]
clsusa1ipm01.internal.cls.com
clsusa1cfg01.internal.cls.com
mon02.cls1.usa1.internal.cls.com
dns01.cls1.usa1.internal.cls.com
dns03.cls1.usa1.internal.cls.com
clsusa1log01.internal.cls.com
clsusa1log03.internal.cls.com
clsusa1esn02.internal.cls.com
clsusa1mon03.internal.cls.com
clsusa1mon04.internal.cls.com

[group2]
clsusa1jnk01.internal.cls.com
mta01.afg1.nor1.internal.cls.com
mon01.afg1.nor1.internal.cls.com
dns02.afg1.nor1.internal.cls.com
clsusa1log02.internal.cls.com
clsusa1esn01.internal.cls.com
clsusa1esn03.internal.cls.com
clsusa1std01.internal.cls.com

[noreboot]
clsusa1rhs01.internal.cls.com
clsusa1ssh01.internal.cls.com
clsusa1sto01.internal.cls.com
vds01.afg1.nor1.internal.cls.com

[satellite]
clsusa1rhs01.internal.cls.com
```

Inside the `group_vars` and `host_vars` directories reside subdirectories named for either the group or the host for which variables are defined.  Order of precedence prioritizes host definitions over group definitions.  For application environments with no special requirements, a standard `all` group is defined.  `all` group definitions are overridden by more specific group and host definitions.  Following are examples:


*/ansible/inventory/group_vars/all/config*
```
# File name of pre and post patching scripts.
pre_update_script: /opt/home/svc_cls/pre_update.sh
post_update_script: /opt/home/svc_cls/post_update.sh
post_update_conf: /opt/home/svc_cls/post_update_conf.env
env_verify_dest: /opt/home/svc_cls/env_verify.sh

# Comma-separated list of application team email addresses
app_email: app_owners@<yourdomain>.com
```


*/ansible/inventory/host_vars/clsusa1ipm01.intneral.cls.com (the `role_path` variale is globally defined in the Ansible configuration)*
```
env_verify_src: "{{ role_path }}/files/host/clsusa1ipm01.internal.cls.com /env_verify.sh"
pre_update_src: "{{ role_path }}/files/host/clsusa1ipm01.internal.cls.com /pre_update.sh"
```

### Playbooks

The `playbooks` directory houses all the playbooks (written in YAML) that are used to execute automated patching actions, as well as any other automation that is implemented through Ansible.  In the following example, update.yaml includes logic for emailing notifications both before and after patching activities are executed against a host.  The roles included in this playbook are explored later on.



/ansible/playbooks/update.yaml
```
---
#------------------------------------------------------------------------------
# Description: Playbook to perform 'yum update' on selected hosts.
#        NOTE: This playbook will also reboot hosts if kernel is updated.
#------------------------------------------------------------------------------

- name: Performing yum update on host(s)
  hosts: group1
  become: yes
  any_errors_fatal: false

  ##########################################################
  # Send notification email at start of change to app teams
  ##########################################################
  pre_tasks:
    - mail:
        subject: 'OS patching for {{ ansible_hostname }} is beginning now.'
        from: 'svc_cls@{{ ansible_fqdn }}'
        body: 'Automated OS patching for {{ ansible_hostname }} is beginning now.'
        to: '{{ app_email }}'
        charset: utf8
      delegate_to: localhost
      tags: mail

  #########################################################
  # Execute update procedures
  #########################################################
  roles:
    - { role: pre_update, become: yes }
    - { role: yum_update, become: yes }
    - { role: post_update, become: yes }

  ##########################################################
  # Send notifiction of completion
  ##########################################################
  post_tasks:
    - mail:
        subject: 'OS patching for {{ ansible_hostname }} has completed.'
        from: 'svc_cls@{{ ansible_fqdn }}'
        body: "Automated OS patching for {{ ansible_hostname }} has completed."
        to: '{{ app_email  }}'
        charset: utf8
      delegate_to: localhost
      tags: mail 

- name: Performing yum update on host(s)
  hosts: group2
  become: yes
  any_errors_fatal: false

  ##########################################################
  # Send notification email at start of change to app teams
  ##########################################################
  pre_tasks:
    - mail:
        subject: 'OS patching for {{ ansible_hostname }} is beginning now.'
        from: 'svc_cls@{{ ansible_fqdn }}'
        body: 'Automated OS patching for {{ ansible_hostname }} is beginning now.'
        to: '{{ app_email }}'
        charset: utf8
      delegate_to: localhost
      tags: mail

  #########################################################
  # Execute update procedures
  #########################################################
  roles:
    - { role: pre_update, become: yes }
    - { role: yum_update, become: yes }
    - { role: post_update, become: yes }

  ##########################################################
  # Send notifiction of completion
  ##########################################################
  post_tasks:
    - mail:
        subject: 'OS patching for {{ ansible_hostname }} has completed.'
        from: 'svc_cls@{{ ansible_fqdn }}'
        body: "Automated OS patching for {{ ansible_hostname }} has completed."
        to: '{{ app_email  }}'
        charset: utf8
      delegate_to: localhost
      tags: mail 

- name: Performing yum update on host(s)
  hosts: noreboot
  become: yes
  any_errors_fatal: false

  ##########################################################
  # Send notification email at start of change to app teams
  ##########################################################
  pre_tasks:
    - mail:
        subject: 'OS patching for {{ ansible_hostname }} is beginning now.'
        from: 'svc_cls@{{ ansible_fqdn }}'
        body: 'Automated OS patching for {{ ansible_hostname }} is beginning now.'
        to: '{{ app_email }}'
        charset: utf8
      delegate_to: localhost
      tags: mail

  #########################################################
  # Execute update procedures
  #########################################################
  roles:
    - { role: pre_update, become: yes }
    - { role: yum_update, become: yes }
    - { role: post_update, become: yes }

  ##########################################################
  # Send notifiction of completion
  ##########################################################
  post_tasks:
    - mail:
        subject: 'OS patching for {{ ansible_hostname }} has completed.'
        from: 'svc_cls@{{ ansible_fqdn }}'
        body: "Automated OS patching for {{ ansible_hostname }} has completed."
        to: '{{ app_email  }}'
        charset: utf8
      delegate_to: localhost
      tags: mail 

- name: Performing yum update on satellite servers
  hosts: satellite
  become: yes
  any_errors_fatal: false

  ##########################################################
  # Send notification email at start of change to app teams
  ##########################################################
  pre_tasks:
    - mail:
        subject: 'OS patching for {{ ansible_hostname }} is beginning now.'
        from: 'svc_cls@{{ ansible_fqdn }}'
        body: 'Automated OS patching for {{ ansible_hostname }} is beginning now.'
        to: '{{ app_email }}'
        charset: utf8
      delegate_to: localhost
      tags: mail

  #########################################################
  # Execute update procedures
  #########################################################
  roles:
    - { role: pre_update, become: yes }
    - { role: yum_noreboot, become: yes }
    - { role: post_update, become: yes }

  ##########################################################
  # Send notifiction of completion
  ##########################################################
  post_tasks:
    - mail:
        subject: 'OS patching for {{ ansible_hostname }} has completed.'
        from: 'svc_cls@{{ ansible_fqdn }}'
        body: "Automated OS patching for {{ ansible_hostname }} has completed."
        to: '{{ app_email  }}'
        charset: utf8
      delegate_to: localhost
      tags: mail
```

### Roles

Everything inside the `roles` directory and its subdirectories exists to carry out all the actions called out in a playbook.  Basic patching automation makes use of four distinct roles:

* yum_update
* yum_noreboot
* pre_update
* post_update



#### yum_update
`/ansible/roles/yum_update/tasks/main.yaml` executes a yum update to the latest available version on all installed packages, and then performs a check to see if a new kernel was installed, creating a flag that triggers a reboot during `post_update` if one was.


*/ansible/roles/yum_update/tasks/main.yaml*

```
---
#------------------------------------------------------------------------------
# Description: Perform yum update on selected hosts and compare running
#              kernel version with last updated kernel version.
#       Notes: shell module used to compare last kernel and
#              current, can't find a better method.
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
      if [[ -f /opt/home/svc_cls/pre_reboot.sh ]]; then
        /opt/home/svc_cls/pre_reboot.sh
      fi
    fi
  tags:
    - skip_ansible_lint
```



#### yum_noreboot
`/anbsible/roles/yum_noreboot/tasks/main.yaml`executes a yum update to the latest available version on all installed packages.  This role does NOT check for a new kernel or create a flag to trigger a reboot during `post_update`.  This functionality is reserved for those servers and application environments where an automated reboot may cause issues (e.g. rebooting the Satellite server, which houses all YUM repos, in the middle of a patching cycle).

*/anbsible/roles/yum_noreboot/tasks/main.yaml*

```
---
#------------------------------------------------------------------------------
# Description: Perform yum update on selected hosts and compare running
#              kernel version with last updated kernel version.
#------------------------------------------------------------------------------

- name: Updating all packages
  yum:
    name: '*'
    state: latest
  tags:
    - skip_ansible_lint
```


#### pre_update
The `pre_update` role, like the `yum_update` and `yum_noreboot` roles, also has a `main.yaml` file located in its `tasks` subdirectory.  This role looks for the existence of a pre_update.sh script on the remote host and executes it.  The `pre_update` role is more heavily leveraged when an application has special requirements.

*/ansible/roles/pre_update/tasks/main.yaml*

```
---
#------------------------------------------------------------------------------
# Description: Check for existence of pre_update.sh script and run it if
#              found.
#------------------------------------------------------------------------------
- name: Checking if pre_update.sh script exists
  stat:
    path: /opt/home/svc_cls/pre_update.sh
  register: update_scripts


- name: Running pre update script
  command: sh /opt/home/svc_cls/pre_update.sh
  when: update_scripts.stat.exists == true
  ignore_errors: no
```

#### post_update
The `post_update` roles also contains a `main.yaml` file in its `tasks` directory.  This role checks for the reboot flag (created in the `yum_update` role) and initiates a reboot of the remote host if the flag exists.  Ansible then waits for the remote host to come back online before it checks for and executes the `post_update.sh` script on the remote host (if it exists).  At a minimum, post_update triggers a report of newly installed packages and send the notification that patching is complete.



*/ansible/roles/post_update/tasks/main.yaml*
```
---
# Description: Check for existence of post_update.sh script and if reboot
#              flag has been set; if so, reboot host and wait for restart.
#------------------------------------------------------------------------------
- name: Checking if post_update.sh script exists
  stat:
    path: /opt/home/svc_cls/post_update.sh
  register: update_scripts

- name: Checking if reboot flag exists
  stat:
    path: /tmp/reboot
  register: reboot

- name: Clearing reboot flag
  file:
    path: /tmp/reboot
    state: absent
  when: reboot.stat.exists == true

- name: Rebooting host(s).
  shell: sleep 2 && /sbin/shutdown -r now "Reboot required for updated kernel." && sleep 2
  async: 20
  poll: 0
  when: reboot.stat.exists == true

# This version doesn't freak out about sudo permission issues for non-privileged execution.
- name: Waiting for host(s) to reboot
  wait_for_connection:
    delay: 60
    timeout: 300
  when: reboot.stat.exists == true

- name: Running post update script
  command: sh /opt/home/svc_cls/post_update.sh
  when:
    - update_scripts.stat.exists == true

```


# Results

As a result of simply automating the patching and reboot functions through leveraging Ansible, opportunities for human error were effectively reduced.  While pre-patching and post-patching actions still required varying levels of manual effort, depending on the application environment involved, laying the foundation for automation reclaimed enough time to clear the way for a deeper dive into automation, which will be explored in [Part 2](../ansiblepatchingautomation2/) of this series. 


  -Brian

