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
