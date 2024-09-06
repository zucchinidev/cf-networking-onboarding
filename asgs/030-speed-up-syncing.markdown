---
layout: single
title: Speed up ASG Syncing
permalink: /asgs/speed-up-asg-syncing
sidebar:
  title: "Application Security Groups (ASGs)"
  nav: sidebar-asgs
---

## Assumptions
- You have a CF deployed
- You have completed the iptables-primer track
- You have one
  [proxy](https://github.com/cloudfoundry/cf-networking-release/tree/develop/src/example-apps/proxy)
  app pushed and named appA (the fewer apps you have deployed the better)

## What
In the previous stories you created ASGs and saw them flow through the system.
But, wow, was it annoying to wait for them to update or what? Let's speed up syncing.

## How

🤔 **Update Your Manifest and Redeploy**
1. Update this [policy-server-asg-syncer bosh
   property](https://github.com/cloudfoundry/cf-networking-release/blob/a2ea4e60b4e4ea978da0a55f6d700331f2fdb5f7/jobs/policy-server-asg-syncer/spec#L36-L38)
   to be `5`.
1. Update this [vxlan-policy-agent bosh
   property](https://github.com/cloudfoundry/silk-release/blob/39450e1c1ccdaf4b9c6e2188499e01ddd75a3083/jobs/vxlan-policy-agent/spec#L59-L61)
   to be `5`.
1. Redeploy.

🤔 **Try It Again!**
1. Redo the steps in the [User Workflow story](./user-workflow) about "Using Running ASGs". Time
   how long it takes for the ASGs to get updated this time.

## Expected Result
Your ASGs should update within 10 seconds. Much faster than 2 minutes! Hehe,
isn't it kind of mean that I put these instructions in at the end?

