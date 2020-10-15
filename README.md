LXD Containers:
===============

1. Create a new LXD container that will be used as a template
~~~
lxc launch ubuntu:18.04 template
~~~

2. Open a bash shell in the container
~~~
lxc exec template bash
~~~

3. Disable Coudinit
~~~
rm -rf /etc/cloud/; rm -rf /var/lib/cloud/; rm -rf /etc/netplan/50-cloud-init.yaml
~~~

4. Configure container by adding the following conf file:
~~~
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      addresses:
        - 178.63.236.XXX/26 # set this address to one of the assigned container IP addresses
      routes:
        - to: 0.0.0.0/0
          via: 116.202.196.30
          on-link: true
      nameservers:
        addresses:
          - 213.133.100.100
          - 213.133.99.99
          - 213.133.98.98
~~~

5. Restart the container networking
~~~
netplan apply
~~~

6. Check connection by pinging
~~~
ping aubg.edu
~~~

7. Add a new user and disable password
~~~
adduser ansible --disabled-password
~~~

8. Switch to ansible user and create .ssh folder
~~~
su - ansible
mkdir -p ~/.ssh
~~~

9. Add ansible user to the sudoers list:
~~~
visudo
~~~

10. The following line allows ansible to execute commands as root without providing a password:
~~~
ansible ALL=(ALL) NOPASSWD:ALL
~~~

11. Update system and install packages
~~~
apt update && apt dist-upgrade -y 
~~~

12. Install python
~~~
apt install python
~~~

13. Stop the template and create the image
~~~
lxc stop template
lxc publish template --alias template_image
~~~

14. Delete the template container
~~~
lxc delete template
~~~

# Create the containers

1. Create container:
~~~
lxc init template_image box1
lxc init template_image box2
~~~

2. Set the ip to the container:
~~~
lxc file edit box1/etc/netplan/default-network-config.yaml
lxc file edit box2/etc/netplan/default-network-config.yaml
~~~

3. Start the new container
~~~
lxc start box1
lxc start box2
~~~

4. Set disk quota in the container and set the size
~~~
lxc config device add box1 root disk pool=lxdpool path=/
lxc config device set box1 root size 4GB
~~~

~~~
lxc config device add box2 root disk pool=lxdpool path=/
lxc config device set box2 root size 4GB
~~~

# Install and configure Ansible 

1. Add the Ansible PPA repo and install it
sudo apt-add-repository ppa:ansible/ansible
sudo apt-get update && sudo apt-get install ansible

2. Create a config file
vim ~/.ansible.cfg

3. Tell ansible to use the user "ansible" in remote servers. Make ansible connect to remote servers once per task.
~~~
[defaults]
remote_user=ansible

[ssh_connection]
pipelining=true
~~~

4. Add an inventory file
~~~
vim ~/ansible-hosts
~~~

5. Add the ips
~~~
[boxes]
178.63.236.215
178.63.236.216

[boxes:vars]
ansible_python_interpreter=/usr/bin/python3
~~~

6. Start a bash shell and add Ansible's private SSH key to ssh-agent
~~~
ssh-agent bash
ssh-add ~/.ssh/id_rsa
~~~

7. Check ansible ad-hoc command works
~~~
ansible boxes -m ping -i ~/ansible-hosts
~~~

Ansible-DevOps
=========

This role includes three main tasks:
- Update system packages
- Improve the security of the servers
- Install xwiki

Role Variables
--------------

- box_public_keys=~/.ssh/id_rsa.pub (Stores the path to the ssh public keys)
- ufw-allowed-ports=[8080, 80, 22, 443] (Stores the list of ports that the servers will listen to)
- box_packages=[ufw, fail2ban, unattended_upgrades] (Stores the needed packages to be installed)

Dependencies
------------

The main role is divided into two inner roles: one that hardens the security of the servers and the other that installs and configures xwiki.

The main playbook
----------------
The playbook includes the two roles: "secure" and "xwiki". It performs the equivalent of apt update && apt upgrade on the server machines.

    ---
    hosts: boxes
    become: yes
    vars_files:
     - vars.yml
     
    pre-tasks:
     - name: Update cache
       apt: update_cache=yes cache_valid_time=3600 force_apt_get=yes
     
     - name: Perform apt upgrade
       apt: upgrade=dist
       
    roles:
     - secure
     - xwiki
  
Secure role
-----------

A number of tasks are executed in order to harden the security of the servers:

 - Add authorized ssh keys to the server machines
 - Limit super user access to the sudo group
 - Disallow password authentication in SSH
 - Allow ssh only for the main user
 - Disallow root ssh access
 - Delete root password
 - Install ufw, unattended_packages, and jail2ban
 - Allow incoming connections only from specified ports
 
### Adding authorized keys to the servers
The following task adds the public keys of the admin machine to the servers
   ~~~
   - name: Add authorized keys
     authorized_key:
      - user: ansible
      - key: "{{ lookup('file', item) }}"
      with_items: "{{ box_public_keys }}"  
   ~~~
We should allow ssh connection only through key-based authentication. This allows immediate connection to the servers from the admin machine without login. The task uses the "authorized_key" module with two parameters: user and key. This way, we determine the user on the server and the key to transfer. The task uses the "lookup" plugin to find the file where the key is stored. 

### Disallow password authentication in SSH
The following task prevents password authentication in ssh to these servers. Only key-based authentication will be allowed.
   ~~~
   - name: Prevent pass auth
     lineinfile:
      - dest: /etc/ssh/sshd_config
      - regexp: "^PasswordAuthentication"
      - line: "PasswordAuthentication no"
      - state: present
     notify: restart ssh
   ~~~
 The "lineinfile" module enables writing to a file. The "dest" parameter specifies the file. In order to find the line to change, the task uses a regex expression. 
 The "^" symbol tells the task to find the line which starts with whatever comes after "^". 
 In this case, find the line that starts with "PasswordAuthentication". 
 The "line" parameter replaces the content of that line. 
 The 'state' parameter with the value of 'present' tells the task that if the line we are looking for is not found, create it. 
 
 ### Limit super user access to sudo group
     ~~~
     - name: Limit super user access
       command: dpkg-statoveride --update --add root sudo 4750 /bin/su
       register: limit_access
       failed_when: limit_access.rc != 0 and ("already exists" not in limit_access.stderr)
       changed_when: limit_access.rc == 0
     ~~~
The 'dpkg-statoveride' is a management tool. It will set the ownership of a given path. The number 4750 means that the owner can read, write, and execute. The users in the sudo group can only read and execute and the rest can do neither. The result is saved in the "limit_access" through the register module. The 'failed_when' and 'changed_when' tell when the file fails or succeeds.
 
 ### Allow ssh only for the main user
     ~~~
     - name: Allow only the main user
       lineinfile:
        dest: /etc/ssh/sshd_config
        regexp: "^AllowUsers"
        line: "AllowUsers ansible"
        state: present
       notify: restart ssh
     ~~~
 Again the task uses the 'lineinfile' module to change the line where allowed users are defined. In the end, the task also notifies the handler to restart ssh.
 
 ### Disallow root ssh access
      ~~~
       - name: Prevent root ssh access
         lineinfile:
          dest: /etc/ssh/sshd_config
          regexp: "^PermitRootLogin"
          line: "PermitRootLogin no"
          state: present
         notify: restart ssh
     ~~~
The same idea applies here. The task prevents root ssh access.

### Delete root password
     ~~~
       - name: Delete root password
         action: shell passwd -d root
         changed_when: false
     ~~~
Having a root password is a bad idea. This is because the attackers can find workarounds to get root access. By deleting the root password, no one is allowed to login as root. Hence, adding an extra layer of security.

### Install packages
     ~~~
       - name: Install packages
         apt: 
          state: present
          pkg: '{{ box_packages }}'
     ~~~
This task installs three packages: ufw, unattended_upgrades, and fail2ban. The 'ufw' stands for Uncomplicated Firewall. We can use that to allow or prevent certain incoming connections. The 'unattended_upgrades' can be used to remove unused dependencies. Fail2ban is an intrusion prevention software used to prevent from bruteforce attacks.

### Automatically remove unused dependencies
     ~~~
       - name: Remove unused dependencies
         lineinfile:
          dest: /etc/apt/apt.conf.d/50unattended-upgrades
          regexp: "Unattended-Upgrade::Remove-Unused-Dependencies"
          line: "Unattended-Upgrade::Remove-Unused-Dependencies \"true\";"
          state: present
          create: yes
     ~~~
The above tasks uses the 'lineinfile' module to edit a configuration file to remove unused dependencies on the system.

### Adjust APT intervals
     ~~~
       - name: Adjust APT intervals
         copy: src=apt_periodic dest=/etc/apt/apt.conf.d/10periodic
     ~~~
The above task uses the 'copy' module to copy a file in the 'secure' role and send it to the servers to the specified location. The file defines the APT intervals.


### Enable Firewall
     ~~~
       - name: Enable firewall
         ufw: state=enabled policy=deny
     ~~~
The above task enables firewall and denies any traffic by default.

### Enable certain ports
     ~~~
       - name: Enable some ports
         ufw: rule=allow port={{ item }} proto=tcp
         with_items:
          - "{{ ufw_allowed_ports }}"
     ~~~
 The above task allows incoming traffic only from a list of ports. The ports are: 80, 8080, 443, and 22. 
 
 
 The xwiki role
 --------------
 
 The following role installs xwiki on the server machines.
 
 ![image](https://user-images.githubusercontent.com/17888328/96138831-db654680-0f06-11eb-8fdd-506c34a675d1.png)  
The playbook installs xwiki with tomcat9 and mysql.
