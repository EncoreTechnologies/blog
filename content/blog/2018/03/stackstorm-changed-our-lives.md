+++
author = "Nick Maludy"
author_url = "https://github.com/nmaludy"
categories = ["Nick Maludy", "StackStorm", "DevOps", "life"]
date = "2018-03-09"
description = "Discusses how StackStorm has revolutionized the way we write automation code and improve the lives of our DevOps team."
featuredpath = "date"
linktitle = ""
title = "How StackStorm Changed Our Lives"
type = "post"
+++

We started playing around with [StackStorm](https://stackstorm.com/) about 9
months ago just to try out some new tools. After a few months of playing
around we realized that it was much more than a toy and could provide serious
benefits to our DevOps team, our organization and to our customers. Before we
discuss all of the great things about StackStorm, let's start with a little
history.

## History

When I started at Encore in June of 2016 the team had just started working on
an automation platform. The product name of our "old platform" will remain
nameless and will simply be referred to as the "old platform". This job was a
very big transition for me coming from a purely Dev background. This was also my
first experience with enterprise scale IT infrastructure. To boil it down I
didn't know what to expect and had no insight into other tools in the automation
space that could help automate our use case.

Our initial use case for our automation platform was to automate VM provisioning
and retirement. After about a year we started getting requests for generic
automation tasks. The old platform was barely equipped to do VM provisioning
and having it do generic automation tasks was even more painful.

### History - 1) Lack of automated deployments

Very quickly into using the old platform we ran into issues. First and foremost,
the system was not able to be automatically deployed. It came in an OVA format
and once the VM was stood up, all configuration from there on out had to be done
by hand via their Web UI. The average time to setup a single-node instance is
about 8 hours (one day) for an experienced engineer. To setup a 4-node cluster
takes about 3 days. The configuration settings are all stored in the tool's
database making it extremely hard to manage with a config management system.
The lack of automated deployments and config management meant that standing up a
new instance for one of the Devs on our team was a laboring process. Every member
of the team had one, maybe two, instances. If one of the instances got messed up
it was devastating because that meant at least one day worth of effort to go
rebuild it.

### History - 2) Lack of documentation

Secondly, there was a general lack of documentation. To figure out how to use 
any of their internal APIs we had to create a support case. A lot of times 
support would not be able to tell us the right API calls to make or options to 
set, so we would have to resort to `grep`ing their source code (which was a 
mess) followed by a lot of trial and error.

### History - 3) Lack of testing

Third, and one of the largest drawbacks to the old platform, was the complete
lack of ability to test our code. The old platform had no testing framework 
for automation code written for it. There were no plans on the road map to add
a testing framework, and general lack of interest in improving the lives of
the engineers that work on the tool. 

### History - 4) Lack of granular executions

Fourth, along the same lines as the point above, the only way for us to
test a single function/method in the old platform was to run our entire State 
Machine (workflow) from beginning to end. Some of these State Machines are very
complex and can run  anywhere from 1-3 hours. During this time we either had to
be really good at changing gears or just wait for the State Machine execution to
complete or error out. For our VM provisioning State Machine, it took around 1 
hour to run through the entire thing. This means we could only test a maximum of
8 changes a day.

### History - 5) Lack of debugging

Fifth, the only way to have any insight into how the system was executing
our State Machines was to tail the logs on the server. If an error occurred
you might know what function it was in, but generally had no idea what
line it occurred on or what the inputs of the functions were. We ended
up having to print out massive amounts of debug statements to the log files
just so we had a clue as to what was going on. If an engineer forgot to
add logging statements, we were in the dark. This was particularly problematic
when debugging issues in production that were reported some time after
an incident occurs.

The other piece to this was the lack of debugging related to the execution
queue. Again, the only insight you have is to read the log files. If the
queue has a lot of State Machines pending execution you just have to wait
and hope yours gets executed. You also don't know which State Machines
are currently running, all you see in the logs is what functions are being
run. This is not a fun spot to be in when you are trying to debug performance
issues.

### History - Key takeaways

I could go on for days about how painful the old platform was to use, both
in development and in production. I think I'll stop here to focus on how much
better our life is now with StackStorm.

Some key takeaways about the old platform is that it was painful to use,
made our team members lives harder, and slowed down our development velocity.

## Recognizing we had a problem

About a year into working with the old platform, our entire team was fed up.
We realized we were being held back by the old platform. Coming from a Dev
background, I knew we could do better. I also knew that things like Continuous
Integration and automated deployments could change our lives for the better.
We started investigating different alternatives that we could use for automation
and landed on StackStorm. After doing a thorough POC, we got buy-in from 
our management team to make the change.

Our discussions with management revolved around reducing development cost,
increasing development velocity to deliver features faster to customers, and
reducing the number of incidents in productions that was taking away from
our development time. These financial discussions are things that our management
cares about deeply and ultimately resulted in them approving the transition
to StackStorm.

## StackStorm

StackStorm is a very well put together platform designed with developers in 
mind. It breaks automation down into discrete actions and ties these actions
together into workflows that can perform complex operations. StackStorm
also provides a sophisticated event bus that can be used to implement
event-based automation. It has a huge set of existing integrations available
to the community on the [StackStorm exchange](https://exchange.stackstorm.org/),
allowing teams to get started quickly and share their work with the community.
There are all sorts of other goodies and features that StackStorm provides,
for more info I suggest you checkout StackStorm's [website](https://stackstorm.com/)
and [documentation](https://docs.stackstorm.com/) for more info.

### StackStorm - Automated deployments

The first way StackStorm changed our lives was by having automated deployments
available out of the box. Even better, they provide multiple different ways to
automatically deploy StackStorm so teams can choose the method that best fits
their needs. Below is a list of some of the ways that StackStorm can be 
automatically deployed:

- [One-liner install](https://docs.stackstorm.com/install/index.html#ref-one-line-install)
- Vagrant via the [st2vagrant](https://github.com/StackStorm/st2vagrant) repo
- Puppet via the [puppet-st2](https://forge.puppet.com/stackstorm/st2) module (maintained by us here at Encore)
- Chef via the [chef-stackstorm](https://github.com/StackStorm/chef-stackstorm) cookbook
- Ansible via the [ansible-st2](https://github.com/StackStorm/ansible-st2) role and playbook
- Docker via the [st2-docker](https://github.com/StackStorm/st2-docker) repo

Our team primarily uses the `puppet-st2` module primarily because we're a Puppet
shop, but also because we maintain the module and want to "eat our own dogfood".
We also utilize the `st2vagrant` and `one-liner` installs for isolated 
environments, for instance on Dev laptops), so engineers can take their work 
with them.

Each of these install methods takes 3-5 minutes. I personally have probably 5+
Dev instances and another 3 instances on my laptop. If I have a problem with
an instance I quickly tear it down and start over. The process is so quick and
painless I am still taken back.

### StackStorm - Documentation

If I could summarize the [StackStorm documentation](https://docs.stackstorm.com/)
in one word it would be "amazing". Every developer on our team taught themselves
StackStorm by simply reading the documentation and playing around with the
[examples](https://docs.stackstorm.com/start.html#deploy-examples). 
The documentation is packed full of all the information you need from
[quick start](https://docs.stackstorm.com/start.html) to 
[advanced topics](https://docs.stackstorm.com/reference/index.html),
[API documentation](https://api.stackstorm.com/), 
[pack development](https://docs.stackstorm.com/reference/packs.html), 
and even [contributing](https://docs.stackstorm.com/development/index.html)
back to StackStorm itself.

Coming from our old platform, this is such a huge leap forward. We can answer
our own questions and teach others in our organization about how things
are working in a much quicker and more thorough fashion than before.

### StackStorm - Testing

Automation code for StackStorm can be written in any language. Each automation
task is called an `action`, you can organize common actions together into 
a single deployment artifact called a `pack` (short for package). Packs
are distributed as GitHub repos, meaning that by default all code is in
source control. Actions are designed to be concise and composable so they
can be reused in various scenarios in your automation workflows. Unit tests
can be written for Python actions using `unittest2` and are executed with
`nosetest` when contributed to the exchange. We've developed a 
[cookiecutter](https://github.com/audreyr/cookiecutter) template for creating
new packs called [cookiecutter-stackstorm](https://github.com/EncoreTechnologies/cookiecutter-stackstorm)
that includes a `Makefile` with all of the testing framework working out of the
box. See our blog post [automated-creation-of-stackstorm-packs](/blog/2017/10/automated-creation-of-stackstorm-packs/)
for more info in using this cookiecutter template.

As described above, StackStorm makes it super simple to test your automation
code. Even if it's not Python, you can still write unit tests for your isolated
actions in whatever framework and testing harness you like.

The final point about testing is the *extensive* automated testing done by the
StackStorm team on the core project itself. Taking a look at their 
[Makefile](https://github.com/StackStorm/st2/blob/master/Makefile) you will
quickly notice that they're not joking around. They have everything from unit
tests, integration tests, and end-to-end tests. Since all of these components
are tested so heavily, it results in extremely predictable releases. Very rarely
are bugs introduced into the system between releases. When a new bug is found
it is usually the result of a testing gap, and the fix to the bug will include
new tests to ensure the condition is caught in the future.

### StackStorm - Granular execution

StackStorm `actions` are the fundamental unit of automation in the platform.
Actions can be executed individually and are treated like a first-class citizen.
Actions can be tied together to create complex workflows. Workflows themselves
are actually actions that delegate to a workflow back-end to do their processing.
This means that a workflow can be executed just like an action.

The granularity of executions means that when we're developing a new action
we can run it by a simple command `st2 run mypack.myaction input1=value1`. Our 
development velocity increased dramatically with this ability. Now our engineers
can do isolated testing of their action until it's working as expected, 
then tie it into a workflow and execute that workflow individually to ensure
it's working. Workflows can be composed of other workflows so we can test
at multiple levels and isolate failures early in the life cycle.

Granular execution will come into play in our next section about debugging.

### StackStorm - Debugging

Debugging in StackStorm is incredible. Every action execution is recorded in
the database and available via the CLI and Web UI. To see all of the executions
that have occurred you simply run:

```shell
$ st2 execution list -n 5
+----------------------------+------------------+--------------+--------------------------+------------------+---------------+
| id                         | action.ref       | context.user | status                   | start_timestamp  | end_timestamp |
+----------------------------+------------------+--------------+--------------------------+------------------+---------------+
| + 5a9f3a71a814c051653fac13 | examples         | st2admin     | succeeded (10s elapsed)  | Wed, 07 Mar 2018 | Wed, 07 Mar   |
|                            | .mistral-nested  |              |                          | 01:03:45 UTC     | 2018 01:03:55 |
|                            |                  |              |                          |                  | UTC           |
| + 5a9f3c47a814c051653fac24 | examples         | st2admin     | succeeded (10s elapsed)  | Wed, 07 Mar 2018 | Wed, 07 Mar   |
|                            | .mistral-nested  |              |                          | 01:11:35 UTC     | 2018 01:11:45 |
|                            |                  |              |                          |                  | UTC           |
| + 5a9f3c7aa814c051653fac35 | examples         | st2admin     | succeeded (119s elapsed) | Wed, 07 Mar 2018 | Wed, 07 Mar   |
|                            | .mistral-nested  |              |                          | 01:12:26 UTC     | 2018 01:14:25 |
|                            |                  |              |                          |                  | UTC           |
| + 5a9f3d2da814c051653fac46 | examples         | st2admin     | succeeded (8s elapsed)   | Wed, 07 Mar 2018 | Wed, 07 Mar   |
|                            | .mistral-nested  |              |                          | 01:15:25 UTC     | 2018 01:15:33 |
|                            |                  |              |                          |                  | UTC           |
| + 5a9f3ddfa814c051653fac57 | examples         | st2admin     | succeeded (12s elapsed)  | Wed, 07 Mar 2018 | Wed, 07 Mar   |
|                            | .mistral-jinja-  |              |                          | 01:18:23 UTC     | 2018 01:18:35 |
|                            | workbook-complex |              |                          |                  | UTC           |
+----------------------------+------------------+--------------+--------------------------+------------------+---------------+
+----------------------------------------------------------------------------------------------------------------------------+
| Note: Only first 5 action executions are displayed. Use -n/--last flag for more results.                                   |
+----------------------------------------------------------------------------------------------------------------------------+
```

You can quickly see the status of any actions and utilize the `st2 execution get`
command to dive in and see information about an individual execution:

```shell
st2 execution get 5a9f3d2da814c051653fac46
id: 5a9f3d2da814c051653fac46
action.ref: examples.mistral-nested
parameters: 
  cmd: echo -n hello
status: succeeded (8s elapsed)
result_task: nested_workflow_3
result: 
  extra:
    state: SUCCESS
    state_info: null
  stdout: hello3
  tasks:
  - action: core.local
    created_at: '2018-03-07 01:15:31'
    id: 916c1313-b32e-4507-827c-e12e391c2dc2
    input:
      cmd: echo -n hello; echo 3
    name: task1
    published:
      stdout: hello3
    result:
      failed: false
      return_code: 0
      stderr: ''
      stdout: hello3
      succeeded: true
    state: SUCCESS
    state_info: null
    type: ACTION
    updated_at: '2018-03-07 01:15:31'
    workflow_execution_id: b243961d-1194-44a0-b8b8-ee996f07900b
    workflow_name: examples.mistral-basic
start_timestamp: Wed, 07 Mar 2018 01:15:25 UTC
end_timestamp: Wed, 07 Mar 2018 01:15:33 UTC
+-----------------------------+------------------------+-------------------+---------------------+-----------------+
| id                          | status                 | task              | action              | start_timestamp |
+-----------------------------+------------------------+-------------------+---------------------+-----------------+
|   5a9f3d2da814c051653fac49  | succeeded (0s elapsed) | main              | core.local          | Wed, 07 Mar     |
|                             |                        |                   |                     | 2018 01:15:25   |
|                             |                        |                   |                     | UTC             |
| + 5a9f3d2ea814c051653fac4b  | succeeded (2s elapsed) | nested_workflow_1 | examples.mistral-   | Wed, 07 Mar     |
|                             |                        |                   | basic               | 2018 01:15:26   |
|                             |                        |                   |                     | UTC             |
|    5a9f3d2ea814c051653fac4d | succeeded (0s elapsed) | task1             | core.local          | Wed, 07 Mar     |
|                             |                        |                   |                     | 2018 01:15:26   |
|                             |                        |                   |                     | UTC             |
| + 5a9f3d31a814c051653fac4f  | succeeded (1s elapsed) | nested_workflow_2 | examples.mistral-   | Wed, 07 Mar     |
|                             |                        |                   | basic               | 2018 01:15:29   |
|                             |                        |                   |                     | UTC             |
|    5a9f3d31a814c051653fac51 | succeeded (0s elapsed) | task1             | core.local          | Wed, 07 Mar     |
|                             |                        |                   |                     | 2018 01:15:29   |
|                             |                        |                   |                     | UTC             |
| + 5a9f3d33a814c051653fac53  | succeeded (1s elapsed) | nested_workflow_3 | examples.mistral-   | Wed, 07 Mar     |
|                             |                        |                   | basic               | 2018 01:15:31   |
|                             |                        |                   |                     | UTC             |
|    5a9f3d33a814c051653fac55 | succeeded (0s elapsed) | task1             | core.local          | Wed, 07 Mar     |
|                             |                        |                   |                     | 2018 01:15:31   |
|                             |                        |                   |                     | UTC             |
+-----------------------------+------------------------+-------------------+---------------------+-----------------+
```

Where this really pays off is when you have an error in an action or workflow. 
In the following workflow you can see there is an error in the `read_file` task
and quickly determine what the error was. In this case the file doesn't exist:

```shell
[root@stackstorm ~]# st2 run examples.mistral-nested cmd="echo -n hello"
...
id: 5aa01561a814c0622bb6a3c3
action.ref: examples.mistral-nested
parameters: 
  cmd: echo -n hello
status: failed
result_task: read_file
result: 
  failed: true
  return_code: 1
  stderr: 'cat: /tmp/some_crazy_file.txt: No such file or directory'
  stdout: ''
  succeeded: false
error: cat: /tmp/some_crazy_file.txt: No such file or directory
traceback: None
failed_on: read_file
start_timestamp: Wed, 07 Mar 2018 16:37:53 UTC
end_timestamp: Wed, 07 Mar 2018 16:37:59 UTC
+-----------------------------+------------------------+-------------------+---------------------+-----------------+
| id                          | status                 | task              | action              | start_timestamp |
+-----------------------------+------------------------+-------------------+---------------------+-----------------+
|   5aa01561a814c0622bb6a3c6  | succeeded (1s elapsed) | main              | core.local          | Wed, 07 Mar     |
|                             |                        |                   |                     | 2018 16:37:53   |
|                             |                        |                   |                     | UTC             |
| + 5aa01562a814c0622bb6a3c8  | succeeded (2s elapsed) | nested_workflow_1 | examples.mistral-   | Wed, 07 Mar     |
|                             |                        |                   | basic               | 2018 16:37:54   |
|                             |                        |                   |                     | UTC             |
|    5aa01562a814c0622bb6a3ca | succeeded (1s elapsed) | task1             | core.local          | Wed, 07 Mar     |
|                             |                        |                   |                     | 2018 16:37:54   |
|                             |                        |                   |                     | UTC             |
|   5aa01565a814c0622bb6a3cc  | failed (0s elapsed)    | read_file         | core.local          | Wed, 07 Mar     |
|                             |                        |                   |                     | 2018 16:37:57   |
|                             |                        |                   |                     | UTC             |
+-----------------------------+------------------------+-------------------+---------------------+-----------------+
```

We can inspect the `read_file` task by taking string in the `id` column and
running `st2 execution get 5aa01565a814c0622bb6a3cc`:

```shell
$ st2 execution get 5aa01565a814c0622bb6a3cc
id: 5aa01565a814c0622bb6a3cc
status: failed (0s elapsed)
parameters: 
  cmd: cat /tmp/some_crazy_file.txt
result: 
  failed: true
  return_code: 1
  stderr: 'cat: /tmp/some_crazy_file.txt: No such file or directory'
  stdout: ''
  succeeded: false
```

From this output we can quickly see what command was being run (`cat /tmp/some_crazy_file.txt`)
and the error message printed on stderr. To get even more detail about what 
went wrong we can pass in the `--detail` flag:

```shell
$ st2 execution get --detail 5aa01565a814c0622bb6a3cc
+-----------------+--------------------------------------------------------------+
| Property        | Value                                                        |
+-----------------+--------------------------------------------------------------+
| id              | 5aa01565a814c0622bb6a3cc                                     |
| action.ref      | core.local                                                   |
| context.user    | st2admin                                                     |
| parameters      | {                                                            |
|                 |     "cmd": "cat /tmp/some_crazy_file.txt"                    |
|                 | }                                                            |
| status          | failed (0s elapsed)                                          |
| start_timestamp | Wed, 07 Mar 2018 16:37:57 UTC                                |
| end_timestamp   | Wed, 07 Mar 2018 16:37:57 UTC                                |
| result          | {                                                            |
|                 |     "succeeded": false,                                      |
|                 |     "failed": true,                                          |
|                 |     "return_code": 1,                                        |
|                 |     "stderr": "cat: /tmp/some_crazy_file.txt: No such file   |
|                 | or directory",                                               |
|                 |     "stdout": ""                                             |
|                 | }                                                            |
| liveaction      | {                                                            |
|                 |     "runner_info": {                                         |
|                 |         "hostname": "stackstorm.maludy.home",                |
|                 |         "pid": 24982                                         |
|                 |     },                                                       |
|                 |     "parameters": {                                          |
|                 |         "cmd": "cat /tmp/some_crazy_file.txt"                |
|                 |     },                                                       |
|                 |     "action_is_workflow": false,                             |
|                 |     "callback": {                                            |
|                 |         "url":                                               |
|                 | "http://0.0.0.0:8989/v2/action_executions/a5a2e644-a6ee-     |
|                 | 403c-9289-2647d217f906",                                     |
|                 |         "source": "mistral_v2"                               |
|                 |     },                                                       |
|                 |     "action": "core.local",                                  |
|                 |     "id": "5aa01565a814c0622bb6a3cb"                         |
|                 | }                                                            |
+-----------------+--------------------------------------------------------------+
```


Having this capability is invaluable when it comes to determining what went 
wrong during testing or production. Since all inputs and outputs are saved
we can quickly try to reproduce the error ourselves by utilizing the granular
execution capability. In this case we could either use:

```shell
st2 run core.local cmd="cat /tmp/some_crazy_file.txt
```

Or even better we can use the `re-run` capability. This will execute the same
action with the same inputs given a previous execution's ID using the 
`st2 execution re-run <id>` command:

```shell
$ st2 execution re-run 5aa01565a814c0622bb6a3cc
.
id: 5aa01729a814c0622bb6a3ce
status: failed
parameters: 
  cmd: cat /tmp/some_crazy_file.txt
result: 
  failed: true
  return_code: 1
  stderr: 'cat: /tmp/some_crazy_file.txt: No such file or directory'
  stdout: ''
  succeeded: false
```

An even more amazing part of the `re-run` capability is that you can
override parameters on the CLI. So, say for example you have 4 parameters
`a, b, c, d` and want to test out execution the action with 3 of the same
but a different value of `d`, you can do this on the CLI like so:

```shell
st2 re-run execution xyz123 d="some new value"
```

In our example with the `cat` command we can override the command that's
executed like so:

```shell
st2 execution re-run 5aa01565a814c0622bb6a3cc cmd="(test -r /tmp/some_crazy_file.txt && cat /tmp/some_crazy_file.txt) || echo '' "
.
id: 5aa0185ca814c0622bb6a3d4
status: succeeded
parameters: 
  cmd: '(test -r /tmp/some_crazy_file.txt && cat /tmp/some_crazy_file.txt) || echo '''' '
result: 
  failed: false
  return_code: 0
  stderr: ''
  stdout: ''
  succeeded: true
```


### StackStorm - Speed

StackStorm, compared to our old platform, is much much faster at executing
actions and workflows. In our VM provisioning workflow in the old platform,
just the pre-cloning phase took about 10-minutes from receiving the API call
to starting to clone the VM. In StackStorm this now takes about 90 seconds.
Upon further inspection we noticed that the time in the old platform from
receiving the API call to actually starting the provision workflow took
90 seconds. So, in StackStorm we're already beginning to clone the VM
in the same time it would take the old platform to even begin execution.


### StackStorm - Takeaways

I could go on for days about how amazing StackStorm is and what other great
features it provides. Stay tuned to this blog for more posts about what some
of those other features are and how to use them.

Bottom line is that it addressed all of the problems of our old platform
and expanded our automation capabilities by an Extreme amount.


## Our lives today

Today the lives of our developers are so much better. We use and abuse the
automated deployment capabilities of StackStorm to quickly spin up
new instances so we can focus our time on developing automation code and
solving problems for our customers.

We've implemented continuous integration into our software development life 
cycle resulting in consistent, predictable releases. This means less time our 
team spends on responding to incidents and more time coding new features.

When we do need to respond to an incident or debug a problem during development
we can utilize the granular execution model of StackStorm to find and fix
these problems incredibly fast.

In general our engineers spend more time coding on features, delivering value
to ourselves and our customers.


## Going beyond VM provisioning

Going back to our initial use case for our automation platform, it was to
automate VM provisioning and retirement. Now that we've transitioned over to 
StackStorm we're able to go *far* beyond VM provisioned and have started 
automating every part of the stack from networking, to storage, scraping 
power & cooling metrics, website scraping, and business process automation.

These use cases are now growing every day and more parts of our company are
reaping the benefits of migrating to StackStorm as our automation platform.

## Shout out and thank yous

I'd first like to thank the management team here at Encore for having faith
in our team to turn the dream of a generic automation platform into a reality.
With out your support none of this would have been possible.

A huge shout out to the StackStorm team at Extreme Networks for helping and
guiding us along the way. You are hands down the best vendor we've had
the pleasure of working with. The passion and dedication for your work is present
in every release of StackStorm that comes through. Your work has changed
our lives forever.

Finally I'd like to thank the guys on the DevOps team here at Encore.
Thanks for putting up with the old platform for so long. Your hard work
and support during this journey turned our dreams into reality.




Stay tuned to this blog for more write-ups on StackStorm, automation and 
all things DevOps!


-Nick Maludy
