# Ansible Playbook Tutorial with Openstack
This tutorial goes through how to create an ansible playbook and run some basic commands on an instance of stackinabox.
We will use ansible to install some dependencies in order to download and upload to openstack a cirros image.

## Prerequisites
- Install [stackinabox](https://github.com/dcfield/stackinabox.io)
- Install ansible on [windows](https://github.com/dcfield/ansible_on_windows) / [linux](http://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html)

## File setup
- Create a directory eg. `ansible_with_openstack`
- In here, we will have the following files
````
ansible.cfg
inventory
sample_playbook.yaml
````

## /ansible.cfg
- This file has some basic configurations
````
[defaults]
hostfile=inventory
host_key_checking=false

[ssh_connection]
ssh_args = -o ControlMaster=no
````
- The main thing to worry about here is our `hostfile=inventory`
- This will map our host definitions to our `inventory` file

## /inventory
- Here, we define our host, username and password for the openstack machine
- If you have downloaded and started stackinabox, your details will be as follows
````
web1 ansible_ssh_host=openstack.stackinabox.io ansible_ssh_user=demo ansible_ssh_pass=labstack
````
- `web1` is the name we are giving the host. This can be anything you want.
- `ansible_ssh_host` is your host address for openstack
- `ansible_ssh_user` is your login username for openstack
- `ansible_ssh_pass` is your login password for openstack

## /sample_playbook.yaml
Now, let's create our yaml file that will contain all tasks for ansible to perform.

- *Note: Be careful with indentation in yaml files. You must be precise with your spaces!*

### Ansible basics
- First, we begin the file with `---`
- Next, define the hosts. From our `inventory` file, we named our host `web1`
- `- hosts: web1`
- We want to use sudo priviledges, so set `sudo: yes`

### Tasks
- We define our tasks using `tasks:`
- We have four tasks to perform in order to upload our cirros image.
1. Check if the `shade` module is installed. Shade is a python module used for making simple interactions with Openstack.
2. Install shade
3. Download cirros image
4. Upload the cirros image to openstack

#### 1. Check if shade is installed
````
- name: Check if shade module is installed
  command: pip show shade
  ignore_errors: true
  register: shade_installed
  changed_when: false
````

#### 2. Install shade
- This task will only be performed if task 1 failed
````
- block:
    - name: install shade
      command: "pip install shade"
  when: shade_installed.rc != 0
````

#### 3. Download cirros image
````
- name: Download cirros image
  get_url:
      url: http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img
      dest: /tmp/cirros-0.3.4-x86_64-disk.img
````

#### 4. Upload cirros to openstack
````
- name: Upload cirros image to openstack
  os_image:
    auth:
      auth_url: http://192.168.27.100/identity_v2_admin
      password: labstack
      project_domain_id: default
      project_name: admin
      user_domain_id: default
      username: admin
    name: cirros
    container_format: bare
    disk_format: qcow2
    state: present
    filename: /tmp/cirros-0.3.4-x86_64-disk.img
````

### Final file
- Our final file will look like this:
````
---
# Sample ansible playbook
- hosts: web1
  sudo: yes

  tasks:
  - name: Check if shade module is installed
    command: pip show shade
    ignore_errors: true
    register: shade_installed
    changed_when: false

  - block:
      - name: install shade
        command: "pip install shade"
    when: shade_installed.rc != 0

  - name: Download cirros image
    get_url:
      url: http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img
      dest: /tmp/cirros-0.3.4-x86_64-disk.img

  - name: Upload cirros image to openstack
    os_image:
      auth:
        auth_url: http://192.168.27.100/identity_v2_admin
        password: labstack
        project_domain_id: default
        project_name: admin
        user_domain_id: default
        username: admin
      name: cirros
      container_format: bare
      disk_format: qcow2
      state: present
      filename: /tmp/cirros-0.3.4-x86_64-disk.img

````

## Run it
- Run the playbook using `ansible-playbook sample_playbook.yaml`

If successful, you should get an output like the following:

````
PLAY [web1] *************************************************************************************************************************************************************************************

TASK [Check if shade module is installed] ******************************************************************************************************************************************************
ok: [web1]

TASK [install shade] ***************************************************************************************************************************************************************************
skipping: [web1]

TASK [Download cirros image] *******************************************************************************************************************************************************************
changed: [web1]

TASK [Upload cirros image to openstack] ********************************************************************************************************************************************************
changed: [web1]

PLAY RECAP *************************************************************************************************************************************************************************************
web1                        : ok=2    changed=2    unreachable=0    failed=0

````

- In my output, I already have shade installed, so my first task returned ok and ansible skipped the second task

## Check
Go to `http://openstack.stackinabox.io/dashboard/project/images` and check to see if your cirros image has been uploaded. Congrats!
