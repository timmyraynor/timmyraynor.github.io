---
layout: post
title:  "HDF(Hortonworks Data Flow) Automation with Ansible"
date:   2018-05-16 05:53:12 +0000
categories: HDF, Hortonworks, NiFi
---
## NiFi and HDF
NiFi and HDF have become really hot in "Data Streaming", "Data Pipeline" domains. And with the impact from Cloud providers serverless services, having your own on-prim infrastructure and maintain it becomes more and more difficult.

In one of the Big Data project, `Automation` is the keyword to control the maintainence cost of an on-prim HDF cluster within a reasonable range. So far `Ansible` has won a great position in automation orchestration tools hence it is also our choice for automating on-prim HDF cluster deployment.

## Ansible
`Ansible` is famous for the **idempotent** feature as you set your target state in the ansible playbook and ansible will try to reach that target state for you. Main differences between a shell script is you can savely run your ansible playbook multiple times with a progressive delta everytime there's a need of change, which would be very tricky to reach the same goal on Shell scripts.

You could find more about Ansible over [here] or a [quick-start-video]

[quick-start-video]: https://www.ansible.com/resources/videos/quick-start-video
[here]: https://docs.ansible.com/ansible/latest/user_guide/intro_getting_started.html

## Solution Architecture
In the project we breakdown the automation into 4 stages:

- Terraform to create the infrastructure in an idempotent way
- Ansible prepare the Operation System and create the Ambari Cluster
- Ansible submit the Ambari blueprint to create a new HDF cluster (Zero-state-cluster)
- Ansible manage and activate all the configurations after the HDF cluster been created

The following stack chart shows each layer of the platform configurations and automation:
![Automation Stack]({{ "/assets/Automation.png" | absolute_url }})