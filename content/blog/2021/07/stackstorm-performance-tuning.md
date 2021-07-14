+++
author = "Bradley Bishop"
author_url = "https://github.com/bishopbm1"
categories = ["Bradley Bishop", "Stackstorm", "Performance", "Performance Tuning", "Python", "Workflows", "Orquesta"]
date = "2021-07-12"
description = "Looking into Stackstorm performance and how to increase it and the server utilization"
featured = ""
featuredalt = ""
featuredpath = ""
linktitle = ""
title = "Stackstorm Performance Tuning"
type = "post"

+++
# Background

Encore heavily uses all parts of [StackStorm](https://stackstorm.com/) both internally and for our customers to
suit all our automation needs. This includes writing workflows, python files, open and close sourced packs,
and chat ops. We recently had the opportunity to do some performance testing in our environment and we will be discussing
that here.

This post will be focused on a single server deployment looking at both server and code optimizations.
At the time of testing, we are running StackStorm 3.4.1 on RHEL 8 with 8 CPUs and 16gb of RAM virtualized on VMware
installed and configured via Puppet using the [ST2 module](https://forge.puppet.com/modules/stackstorm/st2). All
of the server-side modifications have been added as parameters to this module to help the community scale the
application.

# Establishing a baseline

To start out we need to establish some baselines so that we can have a good starting point to know what performance
gains that we can see and where. With the final goal of running 50 workflows in parallel in 4 minutes.

On a base install of StackStorm with the Puppet Module you will end up with a server with:
- 10x st2actionrunner
- 2x st2api
- 2x st2stream
- 2x st2auth
- 1x st2garbagecollector
- 1x st2notifier
- 1x st2resultstracker
- 1x st2rulesengine
- 1x st2sensorcontainer
- 1x st2chatops
- 1x st2timersengine
- 1x st2workflowengine
- 1x st2scheduler

While I cannot show the actual code for this testing because it was customer focused, I can however
give an overview of tasks and the idea.

This task is to test if a server is pingable after an incident is submitted to an ITSM tool. There are 2 ITSM tools
available and multiple customers that the workflow needs to be able to support via a config. We also need to keep the ITSM tool
up to date as we progress through the workflows. The different tasks are spread across multiple packs. For readability
we attempt to do a lot of the heavy lifting in the Orquesta workflows with individual tasks to make the code reusable and
separate out the data.

The overall structure is below:
```
  config_vars_get                       | monitoring.config_vars_get (Python Action)
  param_override                        | core.noop (Orquesta task)
+ itsm_begin                            | itsm.itsm_incident_update (Orquesta workflow)
    config_vars_get                     | itsm.config_vars_get (Python Action)
    itsm_update                           | itsm.itsm_incident_update (Python Action)
  get_start_timestamp                   | core.noop (Orquesta task)
  ping_test                             | core.local (ST2 Core task)
+ itsm_update_results                   | itsm.itsm_incident_update (Orquesta workflow)
    config_vars_get                     | itsm.config_vars_get (Python Action)
    itsm_update                           | itsm.itsm_incident_update (Python Action)
+ threshold_check                       | monitoring.threshold_check (Orquesta workflow)
    value_check_dispatch                | core.noop (Orquesta task)
    type_dispatch                       | core.noop (Orquesta task)
    verify_above                        | core.noop (Orquesta task)
    verify_below                        | core.noop (Orquesta task)
    sleep_dispatch                      | core.noop (Orquesta task)
    action_recheck                      | core.local (Orquesta task)
  uptime_dispatch                       | core.noop (Orquesta task)
  os_dispatch                           | core.noop (Orquesta task)
  get_uptime_linux                      | core.remote (ST2 Core task)
+ itsm_update_uptime                    | itsm.itsm_incident_update (Orquesta workflow)
    config_vars_get                     | itsm.config_vars_get (Python Action)
    itsm_update                           | itsm.itsm_incident_update (Python Action)
  check_uptime_value                    | core.noop (Orquesta task)
+ itsm_close_uptime                     | itsm.itsm_incident_update (Orquesta workflow)
    config_vars_get                     | itsm.config_vars_get (Python Action)
    itsm_update                           | itsm.itsm_incident_update (Python Action)
  db_dispatch                           | core.noop (Orquesta task)
  get_end_timestamp                     | core.noop (Orquesta task)
+ insert_db                             | monitoring.sql_insert (Orquesta workflow)
    insert_am_process_data              | sql.procedure (Python Action)
    get_timestamps                      | monitoring.convert_timestamp (Python Action)
    merge_metric_dictionaries           | monitoring.merge_dicts (Python Action)
    insert_metrics                      | sql.procedure (Python Action)
    insert_incident_into_metric_details | sql.procedure (Python Action)
    insert_account_into_metric_details  | sql.procedure (Python Action)
    insert_item_into_metric_details     | sql.procedure (Python Action)
    insert_service_into_metric_details  | sql.procedure (Python Action)
  update_kv_var                         | itsm.itsm_processing_incs_var (Python Action)
```

If the ping test fails, the it is possible to attempt to retry 2 more times with a 30 second sleep. Resulting in a total runtime
of 60 to 90 seconds.

Baseline scores:
- 1 workflow: 1m 30s
- 50 workflows: 24m 42s
- 100 workflows: 50m 58s

The main tests we are concerned about is a 50 workflows test that contain 10 unique windows hosts, 15 unique Linux hosts, 25 hosts
that do not exit. And a 100 workflows test that is 2x of the above 50 workflows.

The multiple workflows are timed by the start of the first workflow to the end of the last. Looking at the overall system
usage it was steady a 20-30% for both the CPU and Memory usages. Meaning that limits are being
reached without hitting system limits. We also noticed there are long loading times when loading nested workflows.

# Server Optimizations

First, we will focus on the RHEL system and the StackStorm services. Our goal is to scale out the services that
are being over utilized. The services that we need to focus on are the st2actionrunner, st2workflowengine, and st2scheduler
processes. This is because we are heavy on python actions and Orquesta workflows. We will also need to install a
coordination backend. While there are several options, we will be using Redis. StackStorm does a great job documenting
all of these steps and which ones can be scaled out in their [HA Documentation](https://docs.stackstorm.com/reference/ha.html)

The end result is that you can run multiple instances of these processes and they will run in an active active configuration and share
the load between the processes. To do this a coordination backend is required.

First we need to install Redis and add it to the StackStorm config. We follow the [Stackstorm Coordination Documentation](https://docs.stackstorm.com/coordination.html)
```shell
# Install redis
yum install redis -y

# Edit the StackStorm config
vi /etc/st2/st2.conf

# add the following lines:
[coordination]
url = redis://127.0.0.1:6379
```

Now that we have our coordination backend we can scale out our other services. Ultimately we are shooting to have 20 st2actionrunners,
4 st2workflowengines, and 2 st2scheduler. We chose these numbers because we are Python action and Orquesta workflow heavy and when
scaling out that many items we don't want to overload the scheduler so we have added that to assist in the load. We also tested adding 5
st2workflowengines instead of 4 and running 30 st2actionrunners instead of 20 but what we found is that we started to over load the
host system and things started taking extra time to run.

Scaling out the st2actionrunner is very easy and just need to edit the file: `/etc/sysconfig/st2actionrunner` and change
the worker number per the [Action Runner Documentation](https://docs.stackstorm.com/install/config/config.html#config-configure-actionrunner-workers).
Let’s set this to 20.

Next for the st2workflowengine and st2scheduler we need to do this in a slightly different manor. To scale out these
processes we can simply clone the systems service files.
```shell
# Copy the service files
cp /usr/lib/systemd/system/st2workflowengine.service /usr/lib/systemd/system/st2workflowengine2.service
cp /usr/lib/systemd/system/st2workflowengine.service /usr/lib/systemd/system/st2workflowengine3.service
cp /usr/lib/systemd/system/st2workflowengine.service /usr/lib/systemd/system/st2workflowengine4.service
cp /usr/lib/systemd/system/st2scheduler.service  /usr/lib/systemd/system/st2scheduler2.service

# Start and enable services
systemctl start st2workflowengine2 st2workflowengine3 st2workflowengine4 st2scheduler2
systemctl enable st2workflowengine2 st2workflowengine3 st2workflowengine4 st2scheduler2
```

Then finally restart StackStorm to realize all the different changes that we can make:
```shell
# Restart StackStorm
st2ctl restart
```

Now if you look at all the processes they should look as follows:
- 20x st2actionrunner
- 2x st2api
- 2x st2stream
- 2x st2auth
- 1x st2garbagecollector
- 1x st2notifier
- 1x st2resultstracker
- 1x st2rulesengine
- 1x st2sensorcontainer
- 1x st2chatops
- 1x st2timersengine
- 4x st2workflowengine
- 2x st2scheduler

Now we can retest the workflows:
- 50 workflows: 7m 53s
- 100 workflows: 15m 36s

We can see some great improvements and the server is now utilized at 80%-90% throughout the whole time the
workflows are running.

All of these configuration changes are available with the latest version of the [ST2 module](https://forge.puppet.com/modules/stackstorm/st2).

# Code Optimizations

Now we look at the code. The general idea is to flatten the workflows. Loading a large, nested workflows take
time, so moving these things to python actions can really help out. This made things a little less readable but saw huge performance
improvements by doing this. Also we had some core.noop tasks that were able to be condensed down into the publishes of the previous task.

We made the following changes:
```
config_vars_get | monitoring.config_vars_get (Python Action)                             -> Single python action
param_overrideb | core.noop (Orquesta task)


+ itsm_begin        | itsm.itsm_incident_update (Orquesta workflow)
    config_vars_get | itsm.config_vars_get (Python Action)                               -> Single python action
    itsm_update       | itsm.itsm_incident_update (Python Action)


get_start_timestamp | core.noop (Orquesta task)                                          -> Consolidated to 'config_vars_get' task


+ itsm_update_results | itsm.itsm_incident_update (Orquesta workflow)
    config_vars_get   | itsm.config_vars_get (Python Action)                             -> Single python action
    itsm_update         | itsm.itsm_incident_update (Python Action)



+ threshold_check        | monitoring.threshold_check (Orquesta workflow)
    value_check_dispatch | core.noop (Orquesta task)
    type_dispatch        | core.noop (Orquesta task)
    verify_above         | core.noop (Orquesta task)                                     -> Single python action
    verify_below         | core.noop (Orquesta task)
    sleep_dispatch       | core.noop (Orquesta task)
    action_recheck       | core.local (Orquesta task)

uptime_dispatch | core.noop (Orquesta task)                                              -> Consolidated to single task
os_dispatch     | core.noop (Orquesta task)


+ itsm_update_uptime  | itsm.itsm_incident_update (Orquesta workflow)
    config_vars_get   | itsm.config_vars_get (Python Action)                             -> Single python action
    itsm_update         | itsm.itsm_incident_update (Python Action)


check_uptime_value | core.noop (Orquesta task)                                           -> Consolidated to 'itsm_update_uptime' task


+ itsm_close_uptime  | itsm.itsm_incident_update (Orquesta workflow)
    config_vars_get  | itsm.config_vars_get (Python Action)                              -> Single python action
    itsm_update        | itsm.itsm_incident_update (Python Action)


db_dispatch       | core.noop (Orquesta task)                                            -> Consolidated to single task
get_end_timestamp | core.noop (Orquesta task)


+ insert_db                             | monitoring.sql_insert (Orquesta workflow)
    insert_am_process_data              | sql.procedure (Python Action)
    get_timestamps                      | monitoring.convert_timestamp (Python Action)
    merge_metric_dictionaries           | monitoring.merge_dicts (Python Action)         -> Single python action
    insert_metrics                      | sql.procedure (Python Action)
    insert_incident_into_metric_details | sql.procedure (Python Action)
    insert_account_into_metric_details  | sql.procedure (Python Action)
    insert_item_into_metric_details     | sql.procedure (Python Action)
    insert_service_into_metric_details  | sql.procedure (Python Action)
```

Which leaves us with an end result of:
```
config_vars_get     | monitoring.config_vars_get (Python Action)
itsm_begin          | itsm.itsm_incident_update (Python Action)
ping_test           | core.local (ST2 Core task)
itsm_update_results | itsm.itsm_incident_update (Python Action)
threshold_check     | monitoring.threshold_check (Python Action)
uptime_dispatch     | core.noop (Orquesta task)
get_uptime_linux    | core.remote (ST2 Core task)
itsm_update_uptime  | itsm.itsm_incident_update (Python Action)
itsm_close          | itsm.itsm_incident_update (Python Action)
db_dispatch         | core.noop (Orquesta task)
insert_db           | monitoring.sql_insert (Python Action)
update_kv_var       | itsm.itsm_processing_incs_remove (Python Action)
```

There were 2 types of changes that we made that resulted in python actions. Those were:
1. We had multiple python actions that were strung together by a workflow and combined
them so that they would be a single python action
2. Nested workflows that only had core.noop, or core.local were able to be put into a
python action where all the inputs were passed, and we did the necessary logic in the
python instead.

Orquesta task consolidation was the other big update to the code. Those changes are:
- We started to see diminishing returns when tasks have 5 or more 'next' tasks to
process. So anywhere we could combine tasks into a single task we did.
- Even though sometimes it resulted in duplicate code it was still faster to
moved a single task into the 'next' section of a previous task.

Now we can retest keeping the server optimizations discussed above:
- 50 workflows: 3m 54s
- 100 workflows: 6m 56s

And with those final changes we were able to beat our target number by 6s!

# Conclusion

We learned that we were able to scale out the processes that were getting hit the hardest
and see immediate improvements. That coupled with flattening out the Orquesta workflow
and adding some python actions in there we were able to meet our goal. This flexibility
is one of the main reasons we chose StackStorm and are always learning new ways of
improving the application.
