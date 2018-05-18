---
layout: single
title:  "Developing Ansible Custom Module 101"
date:   2018-05-18 11:53:12 +0000
categories: 
  - Ansible
tags:
  - Ansible
  - Automation
comments: true
---
# Why Repeat This
To be honest, I understand there are tones of tutorials (including the [Official one]) out there talking thoroughly about how to do Ansible modules. For what you will get here, is a quick **get start** tutorial and "I just want to have a module built".

[Official one]: http://docs.ansible.com/ansible/latest/dev_guide/developing_modules.html

## Ready for an Ansible Module?
As we always need to check before move on:

- Do you really need a Module or a playbook/role
- Do you really need a Module or a plugin

Ok, if the above answer is: I need an Ansible Module, then you must needs:

    An idempotent way to deal with my automation process (otherwise I will *shell/command* it)

## What we need for an Ansible Module
You have used Ansible before, and you probably get used to the Ansible concepts and had a guess about what you need for your module, here I am taking an example of when I try to develop an [Ansible Ambari Configure module] via Ambari API + Python, so I need:

- A way to get the parameters from the playbook yaml
- A way to tell the current state (e.g. query current Ambari cluster status via API)
- A way to do my logics of automation (e.g. API call with the parameters passed in)
- A way to tell Ansible: did I changed anything, and what I've changed

It should look like this in the playbook:

    ambari_cluster_config:
        host: localhost
        port: 8080
        username: admin
        password: admin
        cluster_name: my_cluster
        config_type: admin-properties
        ignore_secret: true
        timeout_sec: 10
        config_map:
          db_root_user:
            value: root


So let's get started!

[Ansible Ambari Configure module]: https://github.com/timmyraynor/ansible-ambari-config-module

### Get Those Parameters
Ansible provide a straight forward **new Python Module**, they call it the Ansiballz framework. But translate into code, you need to import a module like below:

    from ansible.module_utils.basic import AnsibleModule

The `AnsibleModule` above provides the lifecycle of extracting yaml playbook parameters and all fancy operations like parsing an input like `value: {{lookup('template','./files/mytemplate'}}`, then it execute the custom logic you specified and gather the change status for you in the end. It also attached few global options through like:

- `_ansible_no_log`
- `_ansible_debug`
- `_ansible_diff`
...

So how we initiate the module? You just need to tell the module what parameters you are expecting. Like below from my custom Ansible Ambari module:

    argument_spec = dict(
        host=dict(type='str', default=None, required=True),
        port=dict(type='int', default=None, required=True),
        username=dict(type='str', default=None, required=True),
        password=dict(type='str', default=None, required=True, no_log=True),
        cluster_name=dict(type='str', default=None, required=True),
        config_type=dict(type='str', default=None, required=True),
        config_tag=dict(type='str', default=None, required=False),
        ignore_secret=dict(default=True, required=False,
                           choices=[True, False]),
        timeout_sec=dict(type='int', default=10, required=False),
        config_map=dict(type='dict', default=None, required=True)
    )

    module = AnsibleModule(
        argument_spec=argument_spec
    )

Done, your ansible module is initated and the helper function `ansible.module_utils.basic._load_params()` will also be called to gather your stdin parameters and set them into global varibles. How do I get all these declared module?

It's easy:

    p = module.params
    host = p.get('host')
    port = p.get('port')
    username = p.get('username')
    password = p.get('password')
    cluster_name = p.get('cluster_name')
    config_type = p.get('config_type')
    config_tag = p.get('config_tag')
    config_map = p.get('config_map')
    ignore_secret = p.get('ignore_secret')
    connection_timeout = p.get('timeout_sec')

Now you have it. You could also putting Jinja2 template in your yaml like the fancy `lookup` plugin. Ansible will resolve them for you.

### What's My Current Status
Unlike **Terraform**, Ansible does not have any state files - it queries the status on the fly. So you need to create a current status query. In my case, I am just writing an API call to ask the current status of the Ambari Configurations.

With the beatiful `requests` package in Python, I just need to do things like:

    r = get(ambari_url, user, password,
              '/api/v1/clusters/{0}?fields=Clusters/desired_configs'.format(cluster_name), connection_timeout)
    try:
        assert r.status_code == 200
    except AssertionError as e:
        e.message = 'Coud not get cluster desired configuration: request code {0}, \
                    request message {1}'.format(r.status_code, r.content)
        raise
    cluster_config = json.loads(r.content)

And you could read more of these logics in my repo.

### Do Your Logic If Needed
As you retrieve the status you want to verify and possibly change. In my situation, I just loop through the configurations returned from Ambari and compare each of them with my provided configurations in the module.

    for key in cluster_config:
        current_value = cluster_config[key]
        if key in config_map:
            desired_value = config_map[key].get('value')
            if desired_value is not None and (current_value == desired_value or str(current_value).lower() == str(desired_value).lower()):
                # if value matched, put it directly into the map
                result_map[key] = current_value
            else:
                changed = True
                # And you could guess the rest
                # update_cluster_with_new_configurations
                .......

### Tell Ansible: Changed or Not or Failed
This back to the time we create that `AnsibleModule` instance `module`, it has 2 trivial method:

    # For process failure
    module.fail_json(msg="failed message", <kargs you want to display in the -vvv mode>)
    # For process success
    module.exit_json(changed=<True/False>, <kargs you want to display in the -vvv mode>)

Now you are done. Leave the rest to `AnsibleModule`.

## Wrap Up
As a Wrap up, you probably will have something like this in your code:

    from ansible.module_utils.basic import AnsibleModule

    def main():
        argument_spec = dict(
            host=dict(type='str', default=None, required=True),
            port=dict(type='int', default=None, required=True),
            ...
        )

        module = AnsibleModule(
            argument_spec=argument_spec
        )

        # Query your target's status

        # Do your logic about the target's current state vs. goal state

        module.exit_json(changed=<change-or-not>, ...)
        # OR module.fail_json(...)

    if __name__ == '__main__':
        main()

## Where to Put and Use?
You could provide `extra libraries` to you ansible playbook, I just put it into the `ansible.cfg` file under my project root:
    
    [defaults]
     # content of ansible.cfg
     ..... Other Configs ....
     library=./extra_modules

This tells ansible to load extra libraries from the folder `extra_modules`, and you could put your modules there, it should be loaded when you run you playbook.


    


    