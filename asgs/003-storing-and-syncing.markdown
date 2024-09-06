---
layout: single
title: Storing and Syncing ASGs
permalink: /asgs/storing-and-syncing-ASGs
sidebar:
  title: "Application Security Groups (ASGs)"
  nav: sidebar-asgs
---

## Assumptions
- You have a CF deployed
- You have one
  [proxy](https://github.com/cloudfoundry/cf-networking-release/tree/develop/src/example-apps/proxy)
  app pushed and named appA (the fewer apps you have deployed the better)
- You have the public_networks security group bound to all running and staging
  containers

## What
In the first story you learned what ASGs *are*, but how are they implemented
under-the-hood?

## The Non-Dynamic Way
By default Dynamic ASGs _are_ enabled, but when they are not enabled this is how
the ASGs flow through the system. This is provided for historical
context since Dynamic ASGs and their codepath are relatively new.

![image](https://github.com/cloudfoundry/cf-networking-release/blob/develop/docs/asg-enforcement-during-container-create-architecture.png?raw=true)

Iptables rules for non-Dynamic ASGs are only written when a new app container
is created. This means the app needs to be restarted in order for the new ASGs
to take affect. That sucks!

## The Dynamic Way

![image](https://github.com/cloudfoundry/cf-networking-release/blob/develop/docs/dynamic-asg-enforcement-architecture.png?raw=true)

You'll notice that the Dymanic ASG codepath is very similar to the c2c network
policy codepath. Both c2c network policies and ASGs result in iptables rules on
Diego Cells. So when the team implemented Dynamic ASGs they decided to reuse
the c2c architecture that already existed.

## How
We are going to create our own ASG and follow it through the system.

📝 **Make your own ASG with an easy to find IP**
1. Make a file `asg.json`
   ```json
   [
     {
       "protocol": "all",
       "destination": "1.2.3.4"
     }
   ]
   ```

1. Create your very own ASG using this file: `cf create-security-group --help`
1. Bind it to all running apps: `cf bind-running-security-group --help`


🤔 **See how it is stored in the Cloud Controller**

1. Use `cf curl` to call a [v3 CAPI
   endpoint](https://v3-apidocs.cloudfoundry.org/) to list all of the security
   groups. Find your brand new security group.

📝 **Look at the syncer logs**

Let's turn on debug logging and look at what the syncer is doing.

1. Figure out which VM the Policy Server ASG Syncer is running on
   ```
   bosh is --ps | grep policy-server-asg-syncer
   ```
1. Bosh ssh onto that VM.
1. Become root: `sudo su -`
1. Look at the config for the syncer.
   ```
   less /var/vcap/jobs/policy-server-asg-syncer/config/policy-server-asg-syncer.json
   ```
1. Change the log level in the config from "info" to "debug".
1. Restart the syncer process.
   ```
   monit restart policy-server-asg-syncer
   ```
1. Watch the logs.
   ```
   tail -f * /var/vcap/sys/log/policy-server-asg-syncer/policy-server-asg-syncer.stdout.log
   ```
1. 😱 You might've seen something like the following in the logs
   ```
   {
     "timestamp": "2024-09-06T19:15:54.962147693Z",
     "level": "error",
     "source": "cfnetworking.policy-server-asg-syncer",
     "message": "cfnetworking.policy-server-asg-syncer.locket-lock.failed-to-acquire-lock",
     "data": {
       "error": "rpc error: code = AlreadyExists desc = lock-collision",
       "lock": {
         "key": "policy-server-asg-syncer",
         "owner": "9bbc1c96-f033-486b-a8ca-b240bb86a188",
         "type": "lock",
         "type_code": 1
       },
     }
   }
   ```
   This means that you have multiple VMs running with the syncer, but there can
   only be one syncer actually syncing at a time.
1. If you saw the above failed-to-acquire-lock log message, ssh into the other
   VM running a syncer, become root, run `monit stop policy-server-asg-syncer`,
   then come back to this VM where the syncer has debug log level set.

❓ What do you see in the logs? Can you find your ASG referenced?

❓ What do you think the `get-security-groups-last-update-response` log line is
all about? How might it relate to the `skipping-update` log lines?

## Expected Result
Everytime the syncer is restarted -- if it has the lock -- the syncer
will grab all ASGs from Cloud Controller and put them in the Policy Server
database. You should've seen this happen in the logs with the message
`cfnetworking.policy-server-asg-syncer.get-security-groups-retry-loop-succeeded`
and then a list of ASGs, including the one you just created.

After this initial sync, the syncer only updates the ASGs when things change or
new ASGs are added.

## Resources
* [Cloud Foundry Blog: "Updating ASG Definitions Across Thousands of Apps with ZERO Restarts!"](https://www.cloudfoundry.org/blog/updating-asg-definitions-with-zero-restarts/)
* [Application Security Groups Documentation](https://docs.cloudfoundry.org/adminguide/app-sec-groups.html)
* [Typical Application Security Groups](https://docs.cloudfoundry.org/adminguide/app-sec-groups.html#typical-groups)
* ["Taking Security to the Next Level—Application Security Groups" by Abby Kearns](https://blog.pivotal.io/pivotal-cloud-foundry/products/taking-security-to-the-next-level-application-security-groups)
* ["Making sense of Cloud Foundry security group declarations" by Sadique Ali](https://sdqali.in/blog/2015/05/21/making-sense-of-cloud-foundry-security-group-declarations/)
