---
layout: post
title:  "HDF(Hortonworks Data Flow) Automation with Ansible"
date:   2018-05-16 05:53:12 +0000
categories: HDF
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

### Infrastructure Layer
As you could see in the automation stack, our on-prim solution was to built EC2 instances within the company VPCs, hence we are using `Terraform`  to initiate the cluster, `Terraform` could maintain a `terraform state file` to make **idempotent** possible. You could find out more about `Terraform` [here].

[here]: https://www.terraform.io/intro/index.html

### Operation System Preparation & Ambari Cluster creation
#### OS prep
Common OS preparation tasks including:

- YUM packages and repo setup
- Python packages
- Particular components setup (e.g. JDK JCE policy setup as a mandantory tick for HDF installation)

As we were working on AWS, we have something extra like:

- hostname: EC2 instances would naturally come with a generated hostname, if your are utilizing `Route 53` for all the hostname resolving. `Ansible` could be in place to manage all hostname changes automation
- volume mounting: If you have extra `EBS` volumes created for EC2 instances, those volumes need to be formatted and mounted into the instance. This could also be done via the Ansible `mount` module.

#### Ambari Cluster
Ambari is a management tool for you to manage what Hadoop services you want to install and how you want them to be installed into your cluster. A simple architecture graph from the [HWX official website] says a million words:
![Ambari Architecture from https://docs.hortonworks.com/HDPDocuments/Ambari-2.6.0.0/bk_ambari-operations/content/architecture.html]({{ "/assets/architecture_no_gang.png" | absolute_url}})

[HWX official website]: https://docs.hortonworks.com/HDPDocuments/Ambari-2.6.0.0/bk_ambari-operations/content/architecture.html

##### ambari-server setup
Ansible could help install `ambari-server` and `ambari-agent` into your server and agent nodes. But there will be a *GOTCHA* as `ambari-server setup` seems not working in non-interative mode. There was a post talking about using following options:

    --database={{ ambari_server_db_type }} \
    --databasehost={{ ambari_server_db_host }} \
    --databaseport={{ ambari_server_db_port }} \
    --databasename={{ ambari_server_db_name }} \
    --databaseusername={{ ambari_server_db_username }} \
    --databasepassword={{ ambari_server_db_password }}

But this just not doing anything for me.

The solution we have is: **Do the setup yourself!**

All the values via `ambari-server setup` command will go into `<ambari-server conf dir>/ambari.properties` e.g. `/etc/ambari-server/conf/ambari.properties`, hence you could easily deal with the changes in Ansible using `lineinfile` module like below:

    - lineinfile:
        path: /etc/ambari-server/conf/ambari.properties
        regexp: "^ambari-server.user="
        line: "ambari-server.user={{ ambari_server_usr }}"

What about **passwords**? It turns out that Ambari just store them into a file in *plaintext*, so following module will do the trick:

    - name: Prepare the password file
      copy:
        content: "{{ ambari_server_db_password }}"
        dest: /etc/ambari-server/conf/password.dat

    - lineinfile:
      path: /etc/ambari-server/conf/ambari.properties
        regexp: "^server.jdbc.rca.user.passwd="
        line: "server.jdbc.rca.user.passwd=/etc/ambari-server/conf/password.dat"

##### The hostname lookup in Route 53
TBD

After tackling these major issues, your Ambari cluster should run like a charm.


### HDF Cluster deployment
The only component within the Automation stack that is not `idempotent` is the Ambari Blueprint. Ambari Blueprint does not support *re-deployment* and *idempotent*, once you failed in the middle of a blueprint deployment, the most common approach to 'fix' it is to destroy the cluster and recreate them.

Also, the life of an Ambari blueprint will be finished at the end of a cluster deployment. Which means values like:

- passwords
- endpoints
- configurations

When you include those configurations into your Ambari blueprint **host_map.json** file, you won't be able to audit/change those values and reapply them again. Hence we need a higher level application to resolve this, HWX provide a sample Github repository [Ambari blueprint playbook], this playbook demonstrate how to use the `uri` module to upload a blueprint and execute the cluster deployment via Ambari API.

[Ambari blueprint playbook]: https://github.com/hortonworks/ansible-hortonworks

We used a similar approach to reach this goal, with `Anible` on top of the blueprint *templates*, we could use `Ansible` to manage the configurations and passwords. These changes could either be within an `Ansible Vault` file or `Ansible Tower`(or `AWX` for the opensource version).


### HDF Cluster Configuration Management