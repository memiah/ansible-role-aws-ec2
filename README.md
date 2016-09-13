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
       - ../vars/vault.secrets.yml
       - ../vars/config.yml
     roles:
       - vm
   
   - name: Start provisioning on new EC2 instances.
     hosts: staging:!local
     become: yes
     remote_user: centos
     gather_facts: True
     vars_files:
         - ../vars/main.yml
         - ../vars/vault.secrets.yml
         - ../vars/config.yml
     roles:
       - { role: webserver, when: "{{ 'webserver' in rol.split(',') }}" }
       - { role: database, when: "{{ 'database' in rol.split(',') }}" }
```

First playbook declaration should be `local` host with `vm` only in role. In the second one you should set the `hosts` with your current inventory name.

Implementing aws-ec2 role to new projects
-----------------------------------------

In order to prevent the provisioning to the servers declared into the current inventory file, you should exclude the main groups to the second playbook role. If we take (for example) the bizdir project, the `hosts` line in the second playbook role will be:

```
- name: Start provisioning on new EC2 instances.
     hosts: staging:!local:!webservers:!databases
     become: yes
   ...
```

You also must add `local ansible_connection=local` declaration inside the main group(s) into the current inventory file. Following two examples, one for devops-internal-sites project and one for devops-bizdir respectively.

```
[staging]
local ansible_connection=local ansible_python_interpreter=/usr/bin/python

[all:vars]
aws_instance_name='internal-sites-staging-test'
memiah_subdomain_prefix='staging-'
bizdir_database_host='10.0.0.207'
filesystem_s3_backup_bucket='memiah-internal-sites/filesystem-backups'
filesystem_backup_enabled=False
filesystem_restore_enabled=True
```

```
[staging-webservers]
local ansible_connection=local
ec2-52-17-250-117.eu-west-1.compute.amazonaws.com ansible_host=52.17.250.117 private_ip=10.0.0.203 server_density_agent_key=71865781015a009c7ff40bc4559302f8 memiah_wordpress_host=False

[staging-databases]
local ansible_connection=local
ec2-52-16-157-36.eu-west-1.compute.amazonaws.com ansible_host=52.16.157.36 private_ip=10.0.0.207 database_master=True server_density_agent_key=d288e0930b86c2dcf591ecf419e3f6d3

[webservers:children]
staging-webservers

[databases:children]
staging-databases

[staging:children]
webservers
databases

...
```

Declaring hosts templates
-------------------------
A full example can be found in `defaults/main.example.yml`. In `vars/main` of your local project should be declared the `aws_hosts` var (and must be a dictionary).
You can set multiple instances and define which roles you want to provision in each. You **must** declare all roles you want to perform in the main playbook with `when: "{{ 'rolename' in rol.split(',') }}" }` condition (as shown in the example playbook).

```
aws_hosts:
   - aws_instances: 1                           # Number of instances that will be created. AWS will check if this/these instance(s) exists, based on instance name and aws_roles.
     aws_instance_name: 'test-2-devops-webdb'   # The instance(s) name. It will appear into the instance name as `Name` tag
     aws_inventory_tag: True                    # Append automatically the current inventory name to aws_instance_name
     aws_instance_type: t2.medium               # Instance type
 
     aws_region: eu-west-1                      # Instance region
     aws_subnet_id: subnet-db126a82             # Instance subnet ID
 
     aws_security_group: ansible-ec2-test       # AWS security group name. Must be created before
     aws_public_key_name: ben-public-key        # AWS public key name. Must be created before
 
     aws_image_AMI: ami-7abd0209                # AMI ID
 
     aws_volumes:                           # Specify the volumes for new instance(s). Can be empty
       - device_name: /dev/sda1
         volume_type: gp2
         volume_size: 115
         delete_on_termination: true
     aws_roles:                             # Set the roles you want to perform into this/these instance(s). Can be empty
       - webserver
       - database
```

How to run
----------
```
ansible-playbook playbooks/aws_instance.yml --ask-vault-pass -i inventories/staging -vvv
```

Assuming that aws playbook name is `aws_instance.yml`

License
-------

MIT / BSD

Author Information
------------------
This project was created in 2016 by [Memiah Limited](https://github.com/memiah) .
