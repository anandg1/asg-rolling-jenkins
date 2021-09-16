# ASG Rolling Update using Ansible and Jenkins
[![Build Status](https://travis-ci.org/joemccann/dillinger.svg?branch=master)]()

## Description:
I going to demonstrate a rolling update for an auto-scaling group which contains 2 websites using an ansible-playbook with the help of Jenkins & Git , i.e, When ever the developer makes changes to the website and updates changed files in git, it will be updated atomatically in the existing website automatically.
 
## Pre-requesites:

- Basic Knowledge in AWS services, Ansible and Jenkins

## Features:
- ASG Rolling update is possible using ansible-playbook,Jenkins and Git
- No need of an inventory file as we are using Dynamic ASG

 How to Install:
 
 Click to know more about Installation:
 1) [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html)
 2) [Jenkins](https://www.jenkins.io/doc/book/installing/linux/)

 ## Existing Infrastructure:
 An ASG with 2 EC2 instances running which have the same copies of a running website. The load is balanced between them using an Elastic Load Balancer.
 
 ## Possible problems that can occur and their solutions:
 
 1) Every time the developers makes changes to the website and push the new site contents to the git, the ansible playbook has to be run manually inorder to fetch the new files from git and replace it with existing ones in the website so that the changes would be reflected.

Solution is to automate the the process via Jenkins(Continuos Deployment). Once the developer pushes the edited code to Github, Jenkins Server will be triggered via the webhook configured. So that the the build will take place and will be running the ansible-playbook automatically to update the changes to the exisitng site.
 
 2)  When the update happens, instances will be taken out of service and new ones will be created instead of them,  which causes downtime as well as extra billing expense.
 
 The play book is written in such a manner that only one instance will be updated at a time and that too without deleting the existing instance. This is implemented with the help of 'health check' feature of ASG and controlling playbook execution using 'seriel'. The instances would be taken out of ASG one at a time by creating a health check failure artificially and once the changes are made to it, it is moved back to the ASG and put back to health before taking the other instance out of ASG. Thus one instance will be available always which ensures that there is no downtime and also since the instances are not deleted, there won't be any extra billing.
 
 ## Architecture:
![
alt_txt
](https://github.com/anandg1/asg-rolling-jenkins/blob/main/Architecture.png)
 ## Code:
 ```sh
 ---
- name: "Dynamic Invetory Of Autoscaling Group"
  hosts: localhost
  vars:
    access_key: "xxxxxxxxx"       <==== replace xx with your IAM user access_key
    secret_key: "yyyyyyyyy"       <==== replace yy with your IAM user secret_key
    region: "zz-zzzzz-z"          <==== replace zz with your region

  tasks:
    - name: "Fetching details of ASG EC2 Instances"
      ec2_instance_info:
        aws_access_key: "{{ access_key }}"
        aws_secret_key: "{{ secret_key }}"
        region: "{{ region }}"
        filters:
          "tag:aws:autoscaling:groupName": "MyASG"         <==== 'MyASG' is the name of my autoscaling group
          instance-state-name: [ "running"]
      register: asgValue                                   <==== It is a custom register. You can give any name

    - name: "Dynamic Inventory Of ASG"
      add_host:
        name: "{{ item.public_ip_address }}"
        groups: "asg"                                   <==== 'asg' is the name given to dynamic nventory group
        ansible_host: "{{ item.public_ip_address }}"
        ansible_port: 22
        ansible_user: "ubuntu"
        ansible_ssh_private_key_file: "/root/devops.pem"   <==== path to my private keyfile 
        ansible_ssh_common_args: "-o StrictHostKeyChecking=no"
      with_items:
        - "{{ asgValue.instances }}"

- name: "Deploying website from Github"
  hosts: asg
  become: true
  serial: 1
  vars:
    gitRepo: "https://github.com/anandg1/test.git"   <====Replace this with your git repo which contain website files
    cloneDir: "/var/git/"
    docRoot: "/var/www/html/"
    health_time: "25"
  tasks:
    - name: "Installing Apache"
      apt:
        update_cache: true
        name:
          - apache2
          - git
        state: present

    - name: "Clone directory creation"
      file:
        path: "{{ cloneDir }}"
        state: directory

    - name: "Cloning Repo from git"
      git:
       repo: "{{ gitRepo }}"
       dest: "{{ cloneDir }}"
      register: cloneValue

    - name: "Disable HealthCheck"
      when: cloneValue.changed
      file:
        path: "/var/www/html/health.txt"
        state: file
        mode: 0000

    - name: "Off Loading the EC2 From ELB"
      when: cloneValue.changed
      pause:
        seconds: "{{ health_time }}"

    - name: "Copy all files from Clone directory to Document root"
      when: cloneValue.changed
      copy:
        src: "{{ cloneDir }}"
        dest: "{{ docRoot }}"
        remote_src: true
        owner: www-data
        group: www-data

    - name: "Deleting contents of clone directory"
      when: cloneValue.changed
      file:
        state: absent
        path: "{{ cloneDir}}"

    - name: "Insert DNS name"
      when: cloneValue.changed
      lineinfile:
        path: "{{ docRoot }}/index.html"
        line: "{{ ansible_fqdn }}"
        insertafter: <h6>Hostname</h6>

    - name: "Restart/Enabled apache"
      when: cloneValue.changed
      service:
        name: apache2
        state: restarted
        enabled: true

    - name: "Enable HealthCheck"
      when: cloneValue.changed
      file:
        path: "/var/www/html/health.txt"
        state: file
        mode: 0644

    - name: "Loading the EC2 back to ELB"
      when: cloneValue.changed
      pause:
        seconds: "{{ health_time }}"
```
## Git Setup:
1)  In your Git repo, move to settings ---> Webhooks
2) Fill the payload URL section with the below code.
```sh
http://< jenkins-server-IP >:8080/github-webhook/
```
Replace jenkins-server-IP with original IP/

## Jenkins Setup:

1) Open Jenkins using http://< serverIP >:8080 in browser
2) To install ansible plugin, go to Manage Jenkins --> Manage Plugins
3) Search 'Ansible' in search box in 'Available' section.
4) Tick Ansible and complete the installation
5) Navigate to Global Tool Configuration
6) Enter the name and executable path for the ansible. (executebale path can be obatined from 'which ansible' command's result in ansible master node)
7) Now, create new job in Jenkins by clicking 'freestyle project'
8) Now, in the source code management, provide the GitHub Repository name
9) Apply build procedure to invoke the ansible-playbook
10) Update the ansbilbe playbook file location.
11) Provide the vault credentials if the ansible file is already encrypted.
12) Update the Jenkins configuration to trigger the playbook when it recieves webhook by ticking the following
```sh
GitHub hook trigger for GITScm Polling
```
## Conclusion:
Deploying ASG rolling update using GIT and Jenkins has been completed successfully
