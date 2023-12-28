# ansible-do

Create nodes. 

Add nodes to `hosts` file:
```
sudo vim /etc/ansible/hosts
```

```ini
[digital_ocean]
ubuntu-node-01 ansible_host=159.223.178.194
ubuntu-node-02 ansible_host=159.223.98.199

[all:vars]     
ansible_python_interpreter=/usr/bin/python3
```

Running the following command:
```
ansible-inventory --list -y
```

... will return:
```
all:
  children:
    digital_ocean:
      hosts:
        ubuntu-node-01:
          ansible_host: 159.223.178.194
          ansible_python_interpreter: /usr/bin/python3
        ubuntu-node-02:
          ansible_host: 159.223.98.199
          ansible_python_interpreter: /usr/bin/python3
```

## Configure SSH on Control Node

```sh
sudo vim /etc/ssh/ssh_config
```

In `/etc/ssh/ssh_config`, uncomment:
```
   StrictHostKeyChecking accept-new
```


## Configure Hosts

TODO


## Set Up a Repo

Create a folder for the playbooks or clone this repo and change into it.

Create a `vars` directory and within it a `default.yml` file. Add the following: 
```
create_user: jarednielsen

ssh_port: 5995

copy_local_key: "{{ lookup('file', lookup('env','HOME') + '/.ssh/id_rsa.pub') }}"
```

## Secrets

To create a `secret` file, run:
```sh
ansible-vault create secret.yml
```

TODO 
```yml
password: 'password'
password_salt: 'AaBbCcDdEeFf1!2@3#4$5%6^'
```

The `password` is what your user will use to authenticate. Pick a salt that is alphanumeric. 


## Main Playbook

Create an `initial.yml` file in the root of the project and add the following: 
```yml
---

- name: Initial server setup tasks
  hosts: digital_ocean
  remote_user: root
  vars_files:
    - vars/default.yml
    - secret.yml
  tasks:
    - name: update cache
      ansible.builtin.apt:
        update_cache: yes
    - name: Update all installed packages
      ansible.builtin.apt:
        name: "*"
        state: latest
    - name: Make sure NTP service is running
      ansible.builtin.systemd:
        state: started
        name: systemd-timesyncd
    - name: Make sure we have a 'sudo' group
      ansible.builtin.group:
        name: sudo
        state: present
    - name: Create a user with sudo privileges
      ansible.builtin.user:
        name: "{{ create_user }}"
        state: present
        groups: sudo
        append: true
        create_home: true
        shell: /bin/bash
        password: "{{ password | password_hash('sha512', password_salt) }}"
        update_password: on_create
    - name: Set authorized key for remote user
      ansible.posix.authorized_key:
        user: "{{ create_user }}"
        state: present
        key: "{{ copy_local_key }}"
    - name: Disable remote login for root
      ansible.builtin.lineinfile:
        path: /etc/ssh/sshd_config
        state: present
        regexp: ^PermitRootLogin yes
        line: PermitRootLogin no
    - name: Change the SSH port
      ansible.builtin.lineinfile:
        path: /etc/ssh/sshd_config
        state: present
        regexp: ^#Port 22
        line: Port "{{ ssh_port }}"
    - name: UFW - Allow SSH connections
      community.general.ufw:
        rule: allow
        port: "{{ ssh_port }}"
    - name: Brute-force attempt protection for SSH
      community.general.ufw:
        rule: limit
        port: "{{ ssh_port }}"
        proto: tcp
    - name: UFW - Deny other incoming traffic and enable UFW
      community.general.ufw:
        state: enabled
        policy: deny
        direction: incoming
    - name: Remove dependencies that are no longer required
      ansible.builtin.apt:
        autoremove: yes
    - name: Restart the SSH daemon
      ansible.builtin.systemd:
        state: restarted
        name: ssh
- name: Rebooting hosts after initial setup
  hosts: digital_ocean
  port: "{{ ssh_port }}"
  remote_user: "{{ create_user }}"
  become: true
  vars_files:
    - vars/default.yml
    - secret.yml
  vars:
    ansible_become_pass: "{{ password }}"
  tasks:
    - name: Reboot all hosts
      ansible.builtin.reboot: null
...
```

Check your syntax:
```sh
ansible-playbook --syntax-check --ask-vault-pass initial.yml
```

If no errors, run the playbook:
```sh
ansible-playbook --ask-vault-pass initial.yml
```

## 

TODO 







## (Re)Sources

* [How To Automate Initial Server Setup of Multiple Ubuntu 22.04 Servers Using Ansible](https://www.digitalocean.com/community/tutorials/how-to-automate-initial-server-setup-of-multiple-ubuntu-22-04-servers-using-ansible)

