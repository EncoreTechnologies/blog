+++
author = "Nick Maludy"
author_url = "https://github.com/nmaludy"
categories = ["Nick Maludy", "StackStorm", "inquiries", "duo"]
date = "2017-11-01"
description = "Provides a walkthrough of securing a Mistral workflow with Duo 2-factor authentication using ChatOps in StackStorm"
featuredpath = "date"
linktitle = ""
title = "Playing Around With Inquiries In StackStorm 2.5"
type = "post"
+++

[Inquiries](https://docs.stackstorm.com/inquiries.html) are an experimental feature 
in StackStorm 2.5 ([changelog](https://docs.stackstorm.com/changelog.html#october-25-2017))
that allow the workflow to pause and wait for input from a user before proceeding.
In this post we're going to use this new feature to secure a workflow execution
using Duo 2-factor authentication (2FA).

## Getting Started

The code for this post can be downloaded
from the [EncoreTechnologies/stackstorm-demo_inquiries](https://github.com/EncoreTechnologies/stackstorm-demo_inquiries)
GitHub repo. It can be installed using the following command:

```shell
st2 pack install https://github.com/EncoreTechnologies/stackstorm-demo_inquiries.git
```


## Workflow v1

The basic workflow we're going to implement will be restarting a service on the 
StackStorm box:


**actions/service_restart_v1.yaml**
```yaml
---
description: "Restarts a service on the system (without 2FA)"
enabled: true
runner_type: "mistral-v2"
entry_point: workflows/service_restart_v1.yaml
name: service_restart_v1
pack: demo_inquiries
parameters:
  service:
    type: string
    description: "Name of the service to restart."
    required: true
```


**actions/workflows/service_restart_v1.yaml**
```yaml
version: '2.0'

demo_inquiries.service_restart_v1:
  description: Restarts a service on the system (without 2FA)
  type: direct
  input:
    - service

  output:
    result: "{{ _.result }}"
    
  tasks:
    
    service_restart:
      action: core.local_sudo
      input:
        cmd: "systemctl restart {{ _.service }}"
      publish:
        result: "{{ task('service_restart').result }}"
```

Lets test out our workflow by using it to restart our `crond` service:

```shell
$ st2 run demo_inquiries.service_restart_v1 service=crond
..
id: 59f73f65032fb64ea8a99b6b
action.ref: demo_inquiries.service_restart_v1
parameters: 
  service: crond
status: succeeded
result_task: service_restart
result: 
  failed: false
  return_code: 0
  stderr: ''
  stdout: ''
  succeeded: true
start_timestamp: 2017-10-30T15:04:05.821158Z
end_timestamp: 2017-10-30T15:04:09.121116Z
+--------------------------+------------------------+-----------------+-----------------+-----------------+
| id                       | status                 | task            | action          | start_timestamp |
+--------------------------+------------------------+-----------------+-----------------+-----------------+
| 59f73f66032fb64ea8a99b6e | succeeded (1s elapsed) | service_restart | core.local_sudo | Mon, 30 Oct     |
|                          |                        |                 |                 | 2017 15:04:06   |
|                          |                        |                 |                 | UTC             |
+--------------------------+------------------------+-----------------+-----------------+-----------------+
```

This action is very powerful and could potentially cause harm, so in the
following sections we'll detail how to add 2-factor authentication to
this workflow so that services aren't accidentally restarted without
proper approval.


## Hard Coded 2-factor

Our service restart action definitely should not be allowed to run arbitrarily
and we would like to confirm this action prior to executing. To accomplish this
we're going to utilize the StackStorm inquiry feature by insert a new task
in the workflow. This task will execute the `core.ask` action and retrieve
our approval string.

**actions/service_restart_v2.yaml**
```yaml
---
description: "Restarts a service on the system (with a hard coded 2FA check)"
enabled: true
runner_type: "mistral-v2"
entry_point: workflows/service_restart_v2.yaml
name: service_restart_v2
pack: demo_inquiries
parameters:
  service:
    type: string
    description: "Name of the service to restart."
    required: true

```

**actions/workflows/service_restart_v2.yaml**
```yaml
version: '2.0'

demo_inquiries.service_restart_v2:
  description: Restarts a service on the system (with hard coded 2FA)
  type: direct
  input:
    - service

  output:
    result: "{{ _.result }}"
    
  tasks:
    inquiry_2fa:
       action: core.ask
       input:
         route: developers
         schema:
           type: object
           properties:
             secondfactor:
               type: string
               description: "Please enter second factor for restarting the '{{ _.service }}' service"
               required: True
       on-success:
         - service_restart: "{{ task('inquiry_2fa').result.response.secondfactor == 'approved' }}"
              
    service_restart:
      action: core.local_sudo
      input:
        cmd: "systemctl restart {{ _.service }}"
      publish:
        result: "{{ task('service_restart').result }}"
```

We added a new task to this workflow `inquiry_2fa` that executes the `core.ask` 
action. The action takes an input parameter `schema` that defines the format/schema
expected as a response from the inquiry (for other parameter information please see
the `core.ask` documentation [here](https://docs.stackstorm.com/inquiries.html#new-core-ask-action).
This tells StackStorm what information to prompt the user for on the CLI and also
allows StackStorm to validate inquiry responses received via the API against the
defined `schema`.

In this workflow we've hard coded checking the response of our inquiry against
a static string `approved`:

```yaml
       on-success:
         - service_restart: "{{ task('inquiry_2fa').result.response.secondfactor == 'approved' }}"
```


Lets test out our new workflow:

```shell
$ st2 run demo_inquiries.service_restart_v2 service=crond
.
id: 59f75932032fb64ea8a99b87
action.ref: demo_inquiries.service_restart_v2
parameters: 
  service: crond
status: pausing
start_timestamp: 2017-10-30T16:54:10.727011Z
end_timestamp: None
+--------------------------+---------+-------------+----------+-----------------+
| id                       | status  | task        | action   | start_timestamp |
+--------------------------+---------+-------------+----------+-----------------+
| 59f75933032fb64ea8a99b8a | pending | inquiry_2fa | core.ask | Mon, 30 Oct     |
|                          |         |             |          | 2017 16:54:11   |
|                          |         |             |          | UTC             |
+--------------------------+---------+-------------+----------+-----------------+
```

As you can see this executed our workflow but put it into a `pending` state. 
The `core.ask` action paused the workflow and is awaiting a response to the inquiry
we've defined. In order to respond to the inquiry we use the `st2 inquiry` command.


List all of the pending inquiries:

```shell
$ st2 inquiry list
+--------------------------+-------+-------+------------+------+
| id                       | roles | users | route      | ttl  |
+--------------------------+-------+-------+------------+------+
| 59f75933032fb64ea8a99b8a |       |       | developers | 1440 |
+--------------------------+-------+-------+------------+------+
```

Now we need to respond to our inquiry by entering the string `approved`:

**Note: The inquiry ID is the same as the task ID in the workflow**
```shell
$ st2 inquiry respond 59f75933032fb64ea8a99b8a
secondfactor: approved

 Response accepted. Successful response data to follow...
+----------+--------------------------------+
| Property | Value                          |
+----------+--------------------------------+
| id       | 59f75933032fb64ea8a99b8a       |
| response | {                              |
|          |     "secondfactor": "approved" |
|          | }                              |
+----------+--------------------------------+
```

Responding will have resumed our workflow, so lets check on the status:

```shell
$ st2 execution get 59f75932032fb64ea8a99b87
id: 59f75932032fb64ea8a99b87
action.ref: demo_inquiries.service_restart_v2
parameters: 
  service: crond
status: succeeded (133s elapsed)
result_task: service_restart
result: 
  failed: false
  return_code: 0
  stderr: ''
  stdout: ''
  succeeded: true
start_timestamp: 2017-10-30T16:54:10.727011Z
end_timestamp: 2017-10-30T16:56:23.626183Z
+--------------------------+------------------------+-----------------+-----------------+-----------------+
| id                       | status                 | task            | action          | start_timestamp |
+--------------------------+------------------------+-----------------+-----------------+-----------------+
| 59f75933032fb64ea8a99b8a | succeeded (1s elapsed) | inquiry_2fa     | core.ask        | Mon, 30 Oct     |
|                          |                        |                 |                 | 2017 16:54:11   |
|                          |                        |                 |                 | UTC             |
| 59f759b5032fb64ea8a99b8c | succeeded (1s elapsed) | service_restart | core.local_sudo | Mon, 30 Oct     |
|                          |                        |                 |                 | 2017 16:56:21   |
|                          |                        |                 |                 | UTC             |
+--------------------------+------------------------+-----------------+-----------------+-----------------+
```

Our workflow finished successfully and the service has been restarted.
This is definitely not the most robust approach as our 2-factor check is comparing
against a hard coded string within the workflow. Instead what we'd like to do
is use [Duo](https://duo.com/) for our 2-factor authentication mechanism.


## Duo

Duo is a cloud-based 2-factor authentication service that's fairly simple and easy
to use.

### Duo - Setup

To get started we need to sign up for an account on their signup page:
[https://signup.duo.com](https://signup.duo.com).

![Duo Signup Page](/img/2017/11/duo_signup_page.jpg)
 
### Duo - Protect an Application
 
Once you've completed the signup and activation process you will be brought to
a screen where we'll add a new application to protect using Duo. In this cas
we want to protect a `Web SDK` application, type this into the search bar (1)
and click `Protect this Application` (2):

![Protect an Application](/img/2017/11/duo_protect_an_application.jpg)

You will then be brought to a screen with your new credentials for this application.
Save these off, we'll be using them later in the Duo pack config.

![Web SDK Details](/img/2017/11/duo_web_sdk_details.jpg)


### Duo - Adding a User

Duo, by default, does not have any users configured for authentication. To add a
new user click Users (1), Add User (2), enter the username (3) and press the Add User
button (4).

![Duo Add Phone](/img/2017/11/duo_add_user_enter.jpg)

Now that the user is added we need to add a phone to their account. Scroll down
on the users page to the `Phones` section and click `Add Phone` (2):

![Duo Add Phone](/img/2017/11/duo_add_phone.jpg)

Fill out the phone details and click the `Add Phone`(1) button:

![Duo Add Phone Details](/img/2017/11/duo_add_phone_enter.jpg)

Once the phone is added it needs to be added, click the `Activate Duo Mobile`(1):

![Duo Activate Phone](/img/2017/11/duo_activate_phone.jpg)

Click `Generate Duo Mobile Activation Code`(1):

![Duo Generate Mobile Activation Code](/img/2017/11/duo_activate_phone_generate.jpg)

Click `Send Instructions by SMS`(1):

![Duo Send SMS](/img/2017/11/duo_activate_phone_send_sms.jpg)


At this time, go to your phone and install the Duo app via your phones native
application store. Once installed click on the link received via SMS, this will
add the account to your phone. You should now be ready to authenticate to Duo.


## Duo Pack

The next step in our process is integrating StackStorm with Duo. To accomplish 
this we'll utilize the [stackstorm-duo](https://github.com/StackStorm-Exchange/stackstorm-duo)
pack available on exchange. First, we must install the pack:

```shell
$ st2 pack install duo
```

Next, we need to configure the pack to point at our Duo account we just setup.
Earlier in this process when we setup the "Protected Application" for `Web SDK`
there were a set of credentials created that we wrote down, we'll be utilizing
them here. 

If you forgot to write them down, you can retrieve them in the Duo admin
console by going to Applications (1), Web SDK (2):

![Duo Application Credentials](/img/2017/11/duo_application_creds.jpg)

Credentials are composed of three pieces of information. The table below provides
a mapping between the name on the Duo page and the name in the `st2` config.

| Duo Name        | Pack Config |
|-----------------|-------------|
| Integration key | auth_ikey   |
| Secret key	  | auth_skey   |
| API hostname	  | auth_host   |

Using this information, configure the Duo pack:

```shell
$ st2 pack config duo
[root@nor1devssd01 ~]# st2 pack config duo
admin_host: 
auth_skey (secret): ****************************************
admin_skey (secret): 
auth_host: api-xxx.duosecurity.com
admin_ikey (secret): 
auth_ikey (secret): ********************
---
Do you want to preview the config in an editor before saving? [y]: n
---
Do you want me to save it? [y]: y
+----------+---------------------------------------------------+
| Property | Value                                             |
+----------+---------------------------------------------------+
| id       | 59f7621d032fb64ea8a99b9c                          |
| pack     | duo                                               |
| values   | {                                                 |
|          |     "admin_host": null,                           |
|          |     "auth_skey": "********",                      |
|          |     "admin_skey": "********",                     |
|          |     "auth_host": "api-xxx.duosecurity.com",       |
|          |     "admin_ikey": "********",                     |
|          |     "auth_ikey": "********"                       |
|          | }                                                 |
+----------+---------------------------------------------------+
```

Make sure to reload the pack config so StackStorm has the latest values
in its cache:

``` shell
$ st2ctl reload --register-configs
Registering content...[flags = --config-file /etc/st2/st2.conf --register-configs]
2017-10-30 13:32:51,369 INFO [-] Connecting to database "st2" @ "127.0.0.1:27017" as user "stackstorm".
2017-10-30 13:32:51,897 INFO [-] =========================================================
2017-10-30 13:32:51,898 INFO [-] ############## Registering configs ######################
2017-10-30 13:32:51,898 INFO [-] =========================================================
2017-10-30 13:32:52,496 INFO [-] Registered 1 configs.
##### st2 components status #####
st2actionrunner PID: 20122
st2actionrunner PID: 20126
st2actionrunner PID: 20133
st2actionrunner PID: 20137
st2actionrunner PID: 20141
st2actionrunner PID: 20146
st2actionrunner PID: 20154
st2actionrunner PID: 20157
st2actionrunner PID: 20160
st2actionrunner PID: 20164
st2api PID: 20069
st2api PID: 20136
st2stream PID: 20082
st2stream PID: 20156
st2auth PID: 20315
st2auth PID: 20388
st2garbagecollector PID: 20200
st2notifier PID: 20056
st2resultstracker PID: 20098
st2rulesengine PID: 20285
st2sensorcontainer PID: 20234
st2chatops PID: 21248
mistral-server PID: 19978
mistral-api PID: 19977
mistral-api PID: 20009
mistral-api PID: 20010
```

To validate that the config we entered works properly we can check our
authentication credentials using the `duo.auth_check` action:

```shell
$ st2 run duo.auth_check
..
id: 59f7626b032fb64ea8a99b9e
status: succeeded
parameters: None
result: 
  exit_code: 0
  result:
    time: 1509384814
  stderr: ''
  stdout: ''
```

Finally we'll test out actually authenticating using the `duo.auth_auth` action.
We'll be authenticating using a `passcode`, to generate a passcode open up your
Duo mobile app and click the key icon (1) next to your account name. This will
pop up a 6-digit number, that is your passcode.

Using the passcode you just generated run the `duo.auth_auth` action. The username
specified is the username of the account created previously, not your admin account:

```shell
$ st2 run duo.auth_auth username=testuser factor=passcode passcode="1234"
.
id: 59f76368032fb64ea8a99ba7
status: succeeded
parameters: 
  factor: passcode
  passcode: 1234
  username: testuser
result: 
  exit_code: 0
  result:
    result: allow
    status: allow
    status_msg: Success. Logging you in...
  stderr: ''
  stdout: ''
```


## Duo 2-factor Workflow

Now that we have Duo setup we're going to use it for our 2FA instead of comparing
against our static string.


**actions/service_restart_v3.yaml**
```yaml
---
description: "Restarts a service on the system (with duo integrated 2FA)"
enabled: true
runner_type: "mistral-v2"
entry_point: workflows/service_restart_v3.yaml
name: service_restart_v3
pack: demo_inquiries
parameters:
  service:
    type: string
    description: "Name of the service to restart."
    required: true
  duo_username:
    type: string
    description: "Duo username to use for 2FA."
    required: true
```

**actions/workflows/service_restart_v3.yaml**
```yaml
version: '2.0'

demo_inquiries.service_restart_v3:
  description: Restarts a service on the system (with duo integrated 2FA)
  type: direct
  input:
    - service
    - duo_username

  output:
    result: "{{ _.result }}"
    
  tasks:
    inquiry_2fa:
       action: core.ask
       input:
         route: developers
         schema:
           type: object
           properties:
             secondfactor:
               type: string
               description: "Please enter second factor for restarting the '{{ _.service }}' service"
               required: True
       on-success:
         - duo_2fa

    duo_2fa:
      action: duo.auth_auth
      input:
        username: "{{ _.username }}"
        factor: passcode
        passcode: "{{ task('inquiry_2fa').result.response.secondfactor }}"
      on-success:
        - service_restart
              
    service_restart:
      action: core.local_sudo
      input:
        cmd: "systemctl restart {{ _.service }}"
      publish:
        result: "{{ task('service_restart').result }}"
```

Now we'll test out our workflow (v3).

```shell
$ st2 run demo_inquiries.service_restart_v3 service=crond
.
id: 59f772a1032fb64ea8a99bd0
action.ref: demo_inquiries.service_restart_v3
parameters: 
  service: crond
status: pausing
start_timestamp: 2017-10-30T18:42:41.554471Z
end_timestamp: None
+--------------------------+----------------------+-------------+----------+-----------------+
| id                       | status               | task        | action   | start_timestamp |
+--------------------------+----------------------+-------------+----------+-----------------+
| 59f772a2032fb64ea8a99bd3 | running (2s elapsed) | inquiry_2fa | core.ask | Mon, 30 Oct     |
|                          |                      |             |          | 2017 18:42:42   |
|                          |                      |             |          | UTC             |
+--------------------------+----------------------+-------------+----------+-----------------+
```

This time, when responding to the inquiry, we are prompted for both a
`username` and a `passcode`. The `username` is your Duo authentication username
and `passcode` is the 6-digit number generated from the Duo mobile app on your
phone:

``` shell
$ st2 inquiry respond 59f772a2032fb64ea8a99bd3
passcode (secret): ******
username: testuser

 Response accepted. Successful response data to follow...
+----------+----------------------------+
| Property | Value                      |
+----------+----------------------------+
| id       | 59f772a2032fb64ea8a99bd3   |
| response | {                          |
|          |     "passcode": "123456",  |
|          |     "username": "testuser" |
|          | }                          |
+----------+----------------------------+
```

Now that we've responded, lets check on our execution and ensure that it finished
successfully.

``` shell
$ st2 execution get 59f772a1032fb64ea8a99bd0
id: 59f772a1032fb64ea8a99bd0
action.ref: demo_inquiries.service_restart_v3
parameters: 
  service: crond
status: succeeded (41s elapsed)
result_task: service_restart
result: 
  failed: false
  return_code: 0
  stderr: ''
  stdout: ''
  succeeded: true
start_timestamp: 2017-10-30T18:42:41.554471Z
end_timestamp: 2017-10-30T18:43:22.038051Z
+--------------------------+------------------------+-----------------+-----------------+-----------------+
| id                       | status                 | task            | action          | start_timestamp |
+--------------------------+------------------------+-----------------+-----------------+-----------------+
| 59f772a2032fb64ea8a99bd3 | succeeded (1s elapsed) | inquiry_2fa     | core.ask        | Mon, 30 Oct     |
|                          |                        |                 |                 | 2017 18:42:42   |
|                          |                        |                 |                 | UTC             |
| 59f772c4032fb64ea8a99bd5 | succeeded (2s elapsed) | duo_2fa         | duo.auth_auth   | Mon, 30 Oct     |
|                          |                        |                 |                 | 2017 18:43:16   |
|                          |                        |                 |                 | UTC             |
| 59f772c6032fb64ea8a99bd7 | succeeded (1s elapsed) | service_restart | core.local_sudo | Mon, 30 Oct     |
|                          |                        |                 |                 | 2017 18:43:18   |
|                          |                        |                 |                 | UTC             |
+--------------------------+------------------------+-----------------+-----------------+-----------------+
```

## Conclusion

This blog post has demonstrated how to utilize the new inquiries feature
in StackStorm 2.5 to secure sensitive workflows with 2-factor authentication
using Duo. In a future post we'll be looking at how to respond to inquiries
using ChatOps.

-Nick Maludy
