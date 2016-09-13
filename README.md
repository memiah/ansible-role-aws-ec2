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
     hosts: all:!local
     become: yes
     remote_user: centos
     gather_facts: True
     vars_files:
         - ../vars/main.yml
         - ../vars/vault.secrets.yml
         - ../vars/config.yml
     roles:
       - { role: webserver, when: "{{ 'webserver' in role.split(',') }}" }
       - { role: database, when: "{{ 'database' in role.split(',') }}" }
```

First playbook declaration should be `local` host with `vm` only in role.

Declaring the hosts template
----------------------------
You can declare multiple instances and define which roles you want to provision in each VM. You **must** declare all roles you want to perform in the main playbook (example above) with `when: "{{ 'rolename' in role.split(',') }}" }` condition.

```
aws_hosts:
  - instances: 1
    instance_prefix: 'webserver'

    volumes: "{{ aws_volumes }}"    # Just get the default volume
    extra_vars: 'private_ip=10.0.0.203 server_density_agent_key=71865781015a009c7ff40bc4559302f8 memiah_wordpress_host=False'
    roles:
      - webserver

  - instances: 1
    instance_prefix: 'database'
    instance_type: m4.2xlarge

    volumes:
      - device_name: /dev/sda1
        volume_type: gp2
        volume_size: 150
        delete_on_termination: True
    extra_vars: 'private_ip=10.0.0.207 database_master=True server_density_agent_key=d288e0930b86c2dcf591ecf419e3f6d3'
    roles:
      - database
```

`extra_vars` will be passed as raw extra variables in the dynamic-defined new host (in tasks/main, role name `Set the new host inventory`).
If you need different extra_vars for each VM of the same instance, you should declare that instance multiple times instead of rising `instances` number.


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
