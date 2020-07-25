# Guide covers creating an EC2 instance on AWS using Ansible

- To start this guide, go ahead and create a Virtual Machine to host our Ansible (Call this machine ansible helps)

#### Ansible machine Pre-requisites

1. sudo apt update
2. sudo apt install software-properties-common
3. sudo apt-add-repository --yes --update ppa:ansible/ansible
4. sudo apt install ansible 

- You can check ansible is installed with `ansible --version`

## Guide part 1 - Install the EC2 module dependencies

1. sudo apt install python

2. sudo apt install python-pip -y

3. sudo pip install --upgrade pip

4. sudo pip install boto

5. sudo pip install boto3

## Guide part 2 - Create SSH keys to connect to the EC2 instance

- ssh-keygen -t rsa -b 4096 -f ~/.ssh/`<Your_Name>`_aws

## Guide part 3 - Create Ansible Directory Structure

```shell script
mkdir -p AWS_Ansible/group_vars/all/
cd AWS_Ansible
touch playbook.yml
```

## Guide part 4 - Create Ansible Vault file to store the AWS Access and Secret keys

- `ansible-vault create group_vars/all/pass.yml`
- Make your vault password

## Guide part 5 - Edit the vault and store your ec2 secret and access key

- `ansible-vault edit group_vars/all/pass.yml`
- Append to the file. Insert:
```yaml
ec2_access_key: <Your_Access_Key>                                     
ec2_secret_key: <Your_Secret_Key>
```

## Guide part 6 - Open the playbook.yml and add the following

```yaml

# AWS playbook
---

- hosts: localhost
  connection: local
  gather_facts: False

  vars:
    key_name: <Your_Name>_aws #(The key you made earlier)
    region: eu-west-1
    image: # AMI id found on AWS.
    id: "web-app"
    sec_group: "{{ id }}-sec"

  tasks:

    - name: Facts
      block:

      - name: Get instances facts
        ec2_instance_facts:
          aws_access_key: "{{ec2_access_key}}"
          aws_secret_key: "{{ec2_secret_key}}"
          region: "{{ region }}"
        register: result

      - name: Instances ID
        debug:
          msg: "ID: {{ item.instance_id }} - State: {{ item.state.name }} - Public DNS: {{ item.public_dns_name }}"
        loop: "{{ result.instances }}"

      tags: always


    - name: Provisioning EC2 instances
      block:

      - name: Upload public key to AWS
        ec2_key:
          name: "{{ key_name }}"
          key_material: "{{ lookup('file', '~/.ssh/{{ key_name }}.pub') }}"
          region: "{{ region }}"
          aws_access_key: "{{ec2_access_key}}"
          aws_secret_key: "{{ec2_secret_key}}"

      - name: Create security group
        ec2_group:
          name: "{{ sec_group }}"
          description: "Sec group for app {{ id }}"
          # vpc_id: 12345
          region: "{{ region }}"
          aws_access_key: "{{ec2_access_key}}"
          aws_secret_key: "{{ec2_secret_key}}"
          rules:
            - proto: tcp
              ports:
                - 22
              cidr_ip: 0.0.0.0/0
              rule_desc: allow all on ssh port
        register: result_sec_group

      - name: Provision instance(s)
        ec2:
          aws_access_key: "{{ec2_access_key}}"
          aws_secret_key: "{{ec2_secret_key}}"
          key_name: "{{ key_name }}"
          id: "{{ id }}"
          group_id: "{{ result_sec_group.group_id }}"
          image: "{{ image }}"
          instance_type: t2.micro
          region: "{{ region }}"
          wait: true
          count: 1
          instance_tags:
            Name: Eng57.<First>.<L>.WebApp

      tags: ['never', 'create_ec2']
```

- Edit the playbook to fulfil your needs

## Guide part 7 - Create the EC2 Instance

```shell script
ansible-playbook playbook.yml --ask-vault-pass --tags create_ec2
```

## Guide part 8 - SSH into your machine

`ssh -i ~/.ssh/<Your_Name>_aws ubuntu@<Machine_IP>`

#### This concludes our creation of an EC2 instance through Ansible. In the next part we will see how to get the Sample App Working!

# Guide to provision EC2 Instances to host Sample Web App

## Guide part 1 - Launching 2 Instances via Ansible

- Ok so last time we made 1 EC2 instance via Anible, Now we will create 2.
- All we need to do is make an adjustment to the last task. We will create a whole new Task for the DB instance. I have edited the ID of the second to make a new instance rather than replacing the first one
```yaml
      - name: Provision instance WebApp
        ec2:
          aws_access_key: "{{ec2_access_key}}"
          aws_secret_key: "{{ec2_secret_key}}"
          key_name: "{{ key_name }}"
          id: "{{ id }}"
          group_id: "{{ result_sec_group.group_id }}"
          image: "{{ image }}"
          instance_type: t2.micro
          region: "{{ region }}"
          wait: true
          count: 1
          instance_tags:
            Name: Eng57.Samir.D.WebApp.via.Ansible

      - name: Provision instance DB
        ec2:
          aws_access_key: "{{ec2_access_key}}"
          aws_secret_key: "{{ec2_secret_key}}"
          key_name: "{{ key_name }}"
          id: "{{ id }}-2"
          group_id: "{{ result_sec_group.group_id }}"
          image: "{{ image }}"
          instance_type: t2.micro
          region: "{{ region }}"
          wait: true
          count: 1
          instance_tags:
            Name: Eng57.Samir.D.DB.via.Ansible


      tags: ['never', 'create_ec2']
```

