#Memiah Ansible role AWS EC2

Add automatic EC2 instance provisioning via Ansible

##Requirements

Requires Ansible and `boto` python package. You can install it via **PIP**

Role Variables
--------------
In `vars/main.yml` set:
```aws_ec2_provision: True```

Dependencies
------------
No dependencies at this time

Example Playbook
----------------
```
---
   - name: Deploy EC2 instances.
     hosts: local
     become: no
     vars_files:
       - ../vars/main.yml
     roles:
       - ansible-role-aws-ec2

   - name: Start provisioning on new EC2 instances.
     hosts: all:!local
     become: yes
     remote_user: centos
     gather_facts: True
     vars_files:
         - ../vars/main.yml
     roles:
       - { role: webserver, when: "{{ 'webserver' in role.split(',') }}" }
       - { role: database, when: "{{ 'database' in role.split(',') }}" }
```

First playbook declaration should be `local` host with `ansible-role-aws-ec2` only in role.


Example inventory
-----------------

```
[local]
local ansible_connection=local

[staging:children]
local

[another-inventory-group:children]
staging

[staging:vars]
environment_name='test-bizdir'
enviroment_prefix='staging'
```

`enviroment_name` is *mandatory*;
`enviroment_prefix` will be put at the end of the instance name.

Instance name will be composed as following:
```
enviroment_name + instance_prefix + enviroment_prefix
```

Declaring the hosts template
----------------------------
You can declare multiple instances and define which roles you want to provision in each VM. You **must** declare all roles you want to perform in the main playbook (example above) with `when: "{{ 'rolename' in role.split(',') }}" }` condition.


```
aws_hosts:
  - instances: 1
    instance_prefix: 'webserver'
    
    security_group: default
    public_key_name: default

    extra_vars: 'example_var=true'
    roles:
      - webserver

  - instances: 1
    instance_prefix: 'database'
    instance_type: m4.2xlarge
    private_IP: 192.168.1.1

    volumes:
      - device_name: /dev/sda1
        volume_type: gp2
        volume_size: 150
        delete_on_termination: True
    roles:
      - database
```

Normally, you want to declare it in **vars/main.yml**.
`extra_vars` will be passed as raw extra variables in the dynamic-defined new host (in tasks/main, role name `Set the new host inventory`).

If you need different *extra_vars* for each VM of the same instance, **you should declare that instance multiple times** instead of rising `instances` number.
A You can find a full list of available variables in **defaults/main.yml**.


Defaults variables
------------------

```
  environment_name: 'env-name'
  enviroment_prefix: 'prefix'
  
  __aws_instance_type: t2.large
  
  __aws_region: eu-west-1
  __aws_subnet_id: subnet-default
  
  __aws_security_group:
  __aws_public_key_name:
  
  __aws_image_AMI: ami-default
  __aws_private_IP:
  
  __aws_volumes:
    - device_name: /dev/sda1
      volume_type: gp2
  
  __aws_extra_vars:
  aws_tags_check: False
```

All of these vars can be used in the hosts declaration (`aws_hosts` var) for each instance you'll create. Make sure you omit the '_\_aws\_' prefix if exsists.

Role variables
--------------

```
environment_name: 'my-machine'
```

The main name component the instances will have. You can see at the end of the *Example inventory* how the instance(s) name are composed.

```
enviroment_prefix: 'test'
```

An optional prefix that will be put right after the `enviroment_name`

```
aws_tags_check: False
```

This tag enable consistency-test for `Vars` EC2 tag. If that tag doesn't exists on destination instance or is different from the one that would be generated from this Ansible script, it will re-write it.

```
aws_hosts:
  - instances: 1
    instance_prefix: 'webserver'

    extra_vars: 'example_var=true'
    roles:
      - webserver
```

The instance list you want to deploy. See `Declaring the hosts template` section for more info's.

Instances variables
-------------------

```
instances: 1
```

How many instances of the *same type* you want to run. EC2 will recognize automatically instances by comparing the Tags. *Mandatory*


```
instance_prefix: 'webserver'
```

This value will appear in the instance name just after the `enviroment_name`. A `-` character will be put automatically between them.


```
instance_type: t2.large
```

Simply the AWS instance type. You can find a full reference [Here](https://aws.amazon.com/ec2/instance-types/). *Mandatory*


```
region: eu-west-1
```

Where you want to deploy your instance(s). 
Full available regions list available [here](http://docs.aws.amazon.com/general/latest/gr/rande.html#ec2_region).


```
subnet_id: subnet-default
```

You can set the subnet you want your instance will be connected to. 


```
security_group: default
```

AWS security group ID. 


```
public_key_name: public-key
```

You can specify your public key name (Key Pairs). 


```
image_AMI: ami-default
```

The AMI image ID. *Mandatory*


```
private_IP: 172.16.0.1
```

You can specify here a private IP you want to assign to your instance. Make sure that the IP you will enter is not already taken by some other instances in the same subnet.


```
volumes:
  - device_name: /dev/sda1
    volume_type: gp2
    delete_on_termination: True
    encrypted: False

  - devide_name: /dev/sdb1
    volume_type: gp2
```

`volumes` is a list of volumes you want to create with the instance. It follows exactly the same structure as the homonym [Ansible EC2 module](http://docs.ansible.com/ansible/ec2_module.html#options) variable.


```
extra_vars:
    foo: 'baz'
    another_custom_var: 'True'
    full_name: 'John Doe'
```

The `extra_vars` are custom variables you can pass to the Ansible role. All of these variables are then passed as Host vars and they will be available to your tasks eventually.
Following an example:

```
- name: Set system users
  user: name="{{ hostvars[item].vars.foo }}" comment="{{ hostvars[item].vars.full_name }}" uid=1040 group=admin
  when:
    - item != 'local'
    - hostvars[item].vars.foo is defined
  with_items: "{{ groups['databases'] | default([]) }}"
```

How to run
----------
```
ansible-playbook playbooks/aws_instance.yml --ask-vault-pass -i inventories/staging
```

Assuming that AWS playbook name is `aws_instance.yml`

License
-------

MIT / BSD

Author Information
------------------
This project was created in 2016 by [Memiah Limited](https://github.com/memiah) .
