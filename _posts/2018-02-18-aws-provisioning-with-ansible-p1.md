---
layout: post
title: "Using Ansible to Provision AWS Resources - Part 1"
description: "Building EC2 instances with Ansible"
thumb_image: "posts/aws-provisioning-with-ansible/Ansible_Amazon_B_W.png"
tags: [DevOps, Ansible, AWS]
---
NOTE: You can view the repo for this project on
[GitHub](https://github.com/BenCoffeed/AnsibleAWS)

When I took on my first DevOps job, I noticed that while the team had existing CM (Puppet), the process of actually provisioning new resources was extremely manual. I had to dig to find out what AMI's, instance types, security groups, tags, tenancy, VPC structure, subnets, storage configuration, etc. were being used for existing instances. Once I figured all of these things out, I had to figure out what version of puppet was being used. I spent a solid month of just reviewing existing configs and trying to dig through what documentation was available to try and get an idea of how things were operating. I spent all of that time digging through only to realize that the CM we were using had been broken quite a while ago due to the use of a retired custom PHP PPA. Digging through the puppet config, it became quickly evident that the code hadn't been touched in over 3 years. I decided that it would probably be best to simply rewrite the CM and since both the lead backend developer and myself were most familiar with Ansible, we chose to rewrite it using Ansible. I'll post more on that later. . . However, since I was going to be upgrading all of our servers to a current version of Ubuntu as well, I decided to do a blue/green approach and just rebuild everything from scratch. To do this, though, I needed a reliable, repeatable way to build/tear down, and rebuild instances. A wiser man may have used something like HashiCorp's Terraform. However, I didn't see the point of bringing in ANOTHER new technology when I had all the tools to build my infrastructure using Ansible.

## Building EC2 Instances

One thing that I had felt the pain of at the onset of this assignment
was the lack of a consistent inventory of my AWS resources. So, I
decided pretty early on that there was a common list of things that I
would like to know about each instance:
- name
- region
- private ip address
- instance type
- default ssh key name
- security groups
- AMI id
- Does it need a public IP?
- tags
- tenancy
- CloudWatch monitoring
- termination protection
- vpc subnet id
- storage configuration
- ebs optimization
- instance profile
- groups in ansible inventory file
- default ssh username
- location of ssh private key
- default to Ansible using root? (ansible_become)
- host type (ec2, rds, etc. more on this later. . .)
- CloudWatch Alarms
- any pecial variables to add to ansible inventory file

So, I built a simple yml dict that would carry this information for me.
This would be the base of my solution to build new resources. The dict
could hold one or more resources. If the server was a standalone that
would only ever be built by itself, then it would just contain a single
host. However, if it was a cluster of resources that would always need
to be built and destroyed together, then all of the hosts could be
defined in a single dict.

Here's a sample of `awx01.yml`, the inventory file for my Ansible AWX
server.

{% raw %}
```yaml
---
ec2_hosts:
  - name: awx01
    region: us-west-2
    private_ip: 172.17.2.48
    instance_type: t2.medium
    keypair: production
    groups: awx-servers
    image: ami-79873901
    assign_public_ip: yes
    instance_tags:
      Name: awx01
      server_env: utilities
    tenancy: default
    monitoring: yes
    termination_protection: yes
    vpc_subnet_id: subnet-ef8788fd
    wait: yes
    volumes:
      - device_name: /dev/sda1
        volume_size: 40
        delete_on_termination: true
    ebs_optimized: no
    instance_profile_name: AWX-Server-Instance-Profile
    ansible_group: awx-servers
    ansible_ssh_user: ubuntu
    ansible_ssh_private_key_file: .private_keys/production.pem
    ansible_become: true
    host_type: ec2
    enable_alarms: true
    role_vars:
      - awx_db_host: awxdb01
```
{% endraw %}

Now, with that, it's time to add some plays to build ec2 resources. To
do this, I wrote a simple playbook called `provision_resources.yml` that
uses the [Ansible EC2 module](http://docs.ansible.com/ansible/latest/ec2_module.html) to build EC2 resources.

{% raw %}
```yaml
---
- hosts: 127.0.0.1
  connection: local
  vars:
    hashi_vault_token: "{{ lookup('file','.hashi_vault_token') }}"
    hashi_vault_addr: "{{ lookup('env','VAULT_ADDR')}}"

  tasks:
  - include_vars: "{{ inventory }}"

  - name: Provision EC2 hosts
    ec2:
      region: "{{ item.region }}"
      private_ip: "{{ item.private_ip }}"
      instance_type: "{{ item.instance_type }}"
      keypair: "{{ item.keypair }}"
      groups: "{{ item.groups }}"
      image: "{{ item.image }}"
      assign_public_ip: "{{ item.assign_public_ip }}"
      instance_tags: "{{ item.instance_tags | to_json }}"
      tenancy: "{{ item.tenancy }}"
      monitoring: "{{ item.monitoring }}"
      termination_protection: "{{ item.termination_protection }}"
      vpc_subnet_id: "{{ item.vpc_subnet_id }}"
      wait: "{{ item.wait }}"
      volumes: "{{ item.volumes }}"
      ebs_optimized: "{{ item.ebs_optimized }}"
      instance_profile_name: "{{ item.instance_profile_name }}"
    with_items: "{{ ec2_hosts }}"
    aws_access_key: "{{ lookup('hashi_vault', 'secret=aws/creds/production_provision_resources:access_key') }}"
    aws_secret_key: "{{ lookup('hashi_vault', 'secret=aws/creds/production_provision_resources:secret_key') }}"
```
{% endraw %}

Now, to provision the new resource, I simply have to run the command:
`ansible-playbook provision_resources.yml --extra-vars
"inventory=aws_config/ec2/awx01.yml"`

All that's left to do now, is to run my playbook that installs and
provisions Docker and the containers that need to run for my AWX server.
(Unfortunately AWS Fargate isn't available in the us-west-2 region at the
time of the writing of this post or I would be using another server as
an example for this post, but again, more on that later. . . )

However, here's where I hit one of my first snags. By default, the
Python version used on Ubuntu does not allow for Ansible's ssh client to
run its tasks. Luckily, Ansible's [raw module](http://docs.ansible.com/ansible/latest/raw_module.html) allows us to execute an extremely basic ssh command similar to `ssh -c`. Also, before I can run any Ansible plays against this new host, I'll need to make sure that it's actually a member of my hosts inventory. So, I'll also need to make sure my new host exists in my local inventory. Also, if I'm building/rebuilding or changing hosts, I'll need to make sure that there are no issues with my `.ssh/known_hosts`. So, I built another task book that I could import to manipulate my local files.

`aws_mgmt/update_local_files.yml`
{% raw %}
```yaml
---
- name: Add group to ansible inventory
  lineinfile:
    dest: .ansible_hosts
    regexp: '^\[{{ resource_group }}\]'
    line: "[{{ resource_group }}]\n"
    insertbefore: '# Group Configurations'

- name: Add group vars headerto ansible inventory
  lineinfile:
    dest: .ansible_hosts
    regexp: '^\[{{ resource_group }}:vars'
    line: "[{{ resource_group }}:vars]\n"
    insertafter: EOF

- name: Add group vars to ansible inventory
  lineinfile:
    dest: .ansible_hosts
    regexp: '^{{ item }}'
    line: '{{ item }}'
    insertafter: '[{{ resource_group }}:vars]'
  with_items: resource_group_vars
  when: resource_group_vars exists

- name: Add host(s) and key information to ansible inventory under correct group
  lineinfile:
    dest: .ansible_hosts
    regexp: "^{{ resource_ip }}"
    insertafter: '^\[{{ resource_group }}\]'
    line: "{{ resource_ip }} ansible_host={{ resource_name }}  ansible_ssh_private_key_file={{ resource_key_file }} {{ resource_extra_vars | default([]) | join(' ') }} host_type= {{ host_type }}"

- name: Add entry to /etc/hosts
  lineinfile:
    dest: /etc/hosts
    regexp: "{{ resource_name }}"
    line: "{{ resource_ip }}     {{ resource_name }}"
  become: true

- name: Verify that no entries exist in ~/.ssh/known_hosts
  lineinfile:
    dest: ~/.ssh/known_hosts
    regexp: "^{{ resource_name }}"
    line: ""

- name: Verify that no entries exist in ~/.ssh/known_hosts
  lineinfile:
    dest: ~/.ssh/known_hosts
    regexp: "^{{ resource_ip }}"
    line: ""

- name: Reload inventory to pull in changes.
  meta: refresh_inventory
```
{% endraw %}

Then, I just need to update the `post_tasks` in the `provision_resources.yml` playbook to modify my local files and finally update Python.

{% raw %}
```yaml
  post_tasks:
    - name: Update local inventory files
      include: aws_mgmt/update_local_files.yml
        resource_group={{ item.ansible_group }}
        inventory_file={{ inventory_file | default('.ansible_hosts') }}
        resource_ip={{ item.private_ip }}
        resource_name={{ item.name }}
        resource_ssh_user={{ item.ansible_ssh_user }}
        resource_key_file={{ item.ansible_ssh_private_key_file }}
        resource_become={{ item.ansible_become }}
        resource_extra_vars={{ item.role_vars | default([]) }}
        resource_group_vars={{ item.group_vars | when item.group_vars exists }}
      with_items: "{{ ec2_hosts }}"
      when:
        - ec2_hosts is defined
        - item.host_type == 'ec2'

    - name: Wait for host to come up and respond to port 22.
      wait_for:
        host: "{{ item.private_ip }}"
        port: 22
        search_regex: OpenSSH
        delay: 60
        timeout: 300
        state: started
      with_items: "{{ ec2_hosts }}"
      when:
        - ec2_hosts is defined
        - item.host_type == 'ec2'
        - not ec2_repair_mode | default(false)

    - name: Repair apt if in ec2_repair_mode
      delegate_to: "{{ item.private_ip }}"
      raw: sudo apt -y update --fix-missing
      with_items: "{{ ec2_hosts }}"
      when:
        - ec2_hosts is defined
        - item.host_type == 'ec2'
        - ec2_repair_mode | default(false)

    - name: Run post-provisioning tasks
      delegate_to: "{{ item.private_ip }}"
      include: aws_mgmt/post_provision_setup.yml
        resource_ip={{ item.private_ip }}
        resource_name={{ item.name }}
      with_items: "{{ ec2_hosts }}"
      when:
        - ec2_hosts is defined
        - item.host_type == 'ec2'
```
{% endraw %}

You'll also note two extra plays in this inventory. One is a task that
simply waits for port 22 to be available on the newly generated host.
This allows EC2 a chance to actually provision the instance.
Unfortunately, the times for this action vary. So, I added another
variable that can be defined at the run time of the playbook via
`--extra-vars` that will
skip the task of actually provisioning the resource and just tackle the
remote tasks.

The second extra play in the inventory has to do with an issue that I
periodically come across with Ubuntu AMI's where the apt cache is in an
unpredictable state. I used the same `ec2_repair_mode` variable here to
skip the actual provisioning of the host.

That is the basic format for building my EC2 hosts. I'll cover
provisioning RDS hosts in part 2, CloudWatch configuration in Part 3,
and resource termination in part 4.

