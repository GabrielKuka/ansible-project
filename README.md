Ansible-DevOps
=========

This role includes three main tasks:
- Update system packages
- Improve the security of the servers
- Install and configure xwiki

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
 Edit here
 
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
