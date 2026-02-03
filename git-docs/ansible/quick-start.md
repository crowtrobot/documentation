---
description: >-
  Go from nothing to a basically working ansible that can run playbooks as
  quickly as possible
---

# Quick Start

## Step 1 - Understand the Plan

This isn't meant to give you the One True way to use ansible, but a working and good enough starting point, so there's no need to follow along exactly.  You might find you like things arranged differently, and that's fine.  For example, I'll be showing how to create a single inventory file in yaml format, but maybe you'll find that it works better for you to put your inventory into a bunch of small .ini type files in a directory hierarchy.  This is just to get you going, you should expect a lot of this to change as you learn more.  It helps to start with the right expectations, if you're going to learn it well you will need to do a lot of experimenting, and will probably start over a few times.  For that reason we will confine all configs to one directory, so you can have others with other configs without worrying about conflicts. &#x20;



## Step 2 - Install Ansible

There's a lot of weird advice out there, like installing special ansible docker containers and such.  But this is a quick start, so lets not make it any more complicated that it needs to be. &#x20;

With apt:

```
apt install ansible ansible-lint
```

Or with pacman:

```
pacman -S ansible ansible-lint
```

If you plan to use ansible with proxmox, you probably also want to install python-proxmoxer, the python library for accessing the proxmox API.



## Step 3 - Make A Home For Your Ansible Files

I put all my Ansible stuff together in `~/ansible`, but if you are more organized than me, you might want something more sensible, like `~/it_projects/ansible` or something.  It doesn't matter, just modify these steps to match your choice.

```
mkdir ~/ansible
mkdir ~/ansible/playbooks
mkdir ~/ansible/roles
```

And now we will build a basic ansible config in that directory

```
cd ~/ansible
ansible-config init --disabled > ansible.cfg
```

That created a basic config file, with everything set to the default values and commented out.  So open this file with your favorite editor.  For each of these options, find its line in the config file, un-comment it and set it as shown here (they won't all be together as shown here).  We want to keep the option next to its description so it's easy to modify later.  Also this is an .ini type file, so each option has to be in the right section, in this case these are all under the \[defaults] heading

```
[defaults]
inventory = ~/ansible/inventory.yaml
roles_path = ~/ansible/roles
```

At some point you will probably want to start using git to manage version control on your ansible files.  This is expected, so much so that ansible-lint can get confused about the structure of your ansible files if it doesn't find a .git directory, so lets make a quick place-holder one now.

```
mkdir ~/ansible/.git
```

You might notice we only setup the config in this ansible directory.  We could set things up in a more global way, but this way you can try out a different configuration in a different directory.  To make this config take effect, all ansible commands have to be run from this ansible directory.  More on that later. &#x20;

## Step 4 - A Guinea Pig

Ansible usually works by connecting to a remote machine over ssh, and pushing instructions to it.  So we need some machine to try stuff out on.  This can be anything, but I suggest a very basic small VM.  Just do a tiny debian install (or whatever distro you like).  We want this as basic as possible, but with a working ssh server configured to let you log in with your ssh key.  For now this is just a guinea pig for learning on, but it represents what will be the basic machine template you will use.  Think of this as the stem cells of your infrastructure, a basic building block that can be cloned and then specialized into being a specific kind of server (an nginx server, or an nfs server, etc). &#x20;

So make a VM, install just the default debian (or whatever you like) with ssh server.  Once it has booted copy your ssh key to it like this `ssh-copy-id debian@10.10.10.10`.  Make sure you can log in using your ssh key without needing a password, then snapshot this VM so you can revert to this state after trying something out on this machine. &#x20;

You will need to be able to ssh in to root, or ssh in as a user who can then sudo to root without a password.  You will probably want to figure out how to use cloud-init to set the VM's IP on the host before it even boots, but that can come later. &#x20;

FYI - Ansible can work on the same machine it is running on, or it can connect to remote machines in other ways, but we will ignore all that for the moment. &#x20;

## Step 5 - Make Your Inventory

Now we will make an inventory file.  This tells ansible about the machines it will be managing, so it knows what IP to connect to, what username to log in as, and anything else you want it to know about the machines it will be managing.  Every option about every machine in the inventory is basically just a variable and a value (or values since the variable can be a list, or hash or dict, etc). &#x20;

First, lets very briefly talk about how ansible handles variables.  There are lots of places you can set a variable's value, and it is common to set the same variable to different values in different places.  Don't worry for now about all the places they can be set, or the exact order of which of those places take precedence over each other.  For now it is enough to know that more specific takes precedence over the more generic.  So if you set `foo = bar` globally in your inventory but `foo = baz` for a group of machines, then `foo` will be `bar` for all machines, except for those in that group where `foo` will be `baz`.  You could also set `foo = none` for just one machine in that group, which will override the group value, which overrode the global value as you probably already guessed. &#x20;

So with that out of the way, lets make a basic inventory, `~/ansible/inventory.yaml`&#x20;

```
all:
  vars:
    ## Here is where we can set global variables, like these ones
    site_timezone: America/Denver
    ansible_user: debian

  children:
    ## Lets make a group for proxmox host machines, and set some variable specific to them
    proxmox_hosts:
      vars:
        ansible_user: root
      hosts:
        ## and in "hosts" we will define the actual machines, like this one named titanium
        titanium:
          ## and here are the variables specific to this machine
          ansible_host: 192.168.68.22
          case: supermicro_2u
   virtualmachines:
      ## and another group for VMs
      vars:
        proxmox_main_template: 901        
      hosts:
         guinea_pig:
          ansible_host: 10.10.10.10
```

This inventory has 2 sample machines, titanium which is defined as a member of the proxmox\_hosts group, and our virtual machine guinea\_pig which is in the virtual machines group.  Later you will learn how hosts can be in multiple groups (more than just how all of these machines are part of the `all` group), but this is a good starting point to get you up and running. &#x20;

There are a few variables here to note.  The "ansible\_host" variable is an ansible standard variable that tells it what the machine's IP address is (what IP address to ssh to).  There's also ansible\_user which is what user to ssh in to, and as you can see we set that to `debian` globally, but to `root` for all proxmox\_hosts. &#x20;

There's also `site_timezone`, `case`, and `proxmox_main_template`.  These variable don't mean anything to ansible itself, and won't have any effect on anything unless a task, or a role, or a template uses one of those variables for something. &#x20;

* A task is an individual step that ansible can take on a machine, or set of machines (for example, do an apt dist-upgrade). &#x20;
* A role is a set of related tasks to get a machine into some desired state (for example a role might install rsyslog, create a config file that tells it to send logs to the remote logging server, and then starts the rsyslog service). &#x20;
* And a template is text file where variables get replaced with their values as it is copied to the machine being managed.  In our rsyslog role example, the config file that tells syslog to send logs to the remote logging server might get that logging server's IP from a variable set somewhere in the inventory. &#x20;

## Step 6 - Our First Playbook

So finally we can get down to doing some work with ansible.  For this example we will make a playbook that executes a few tasks to install updates.  Create `~/ansible/playbooks/install_updates.yml`

```
- name: Install updates on debian-like systems
##  hosts: all
  hosts: guinea_pig
  become: true
  tasks:
    - name: Only run on debian or 'Ubuntu'
      when: ansible_facts["distribution"] == 'Debian' or ansible_facts["distribution"] == 'Ubuntu'
      block:

        - name: Apt update
          ansible.builtin.apt:
            update_cache: true
            cache_valid_time: 43200

        - name: Auto remove old stuff not needed anymore
          ansible.builtin.apt:
            autoremove: true

        - name: Clean up
          ansible.builtin.apt:
            autoclean: true

        - name: Apply updates
          ansible.builtin.apt:
            upgrade: dist

    - name: Check if reboot required
      ansible.builtin.stat:
        path: /var/run/reboot-required
      register: reboot_required_file

    - name: Tell me if reboot is required
      ansible.builtin.debug:
        msg: This one wants a reboot
      when: reboot_required_file.stat.exists is true and (manual_reboot_only is not defined or manual_reboot_only is true)

    - name: Reboot if required
      ansible.builtin.reboot:
      when: reboot_required_file.stat.exists is true and (manual_reboot_only is defined and manual_reboot_only is not true)

```

This is a pretty simple playbook.  Here is the outline of what it does:

* hosts: - selects what hosts from the inventory it should be applied to (can be all, or a group, or a single host, or a list of hosts).
* become: -Should we sudo when we ssh in?
* Tasks: A list of steps to take.  Most playbooks would actually list roles, but this one is simple. &#x20;
* name ... when ... block: Do this list of tasks if the "when" is true
* And some other tasks to do stuff. &#x20;

So lets run it, from the ansible directory run `ansible-playbook playbooks/install_updates.yaml` and it will take these steps:

* ansible will ssh to the selected hosts, and gather general information about them that might be needed to run later steps (called gathering facts).  These facts are just another variables available to use, called ansible\_facts
* ansible SSHs into the hosts whose facts say they are debian or ubuntu, and tells them to:
  * run the equivalent of an apt update
  * run the equivalent of an apt auto-remove
  * run the equivalent of an apt clean
  * run the equivalent of an apt dist-upgrade
* Check if the /var/run/reboot-required file exists
* If the file exists, and the variable manual\_reboot\_only isn't set on this host, or is set to true, print a messages to tell the user this machine wants to be rebooted
* If the file exists, and the variable manual\_reboot\_only is set to false for this host, start a reboot.  Ansible's reboot command includes automatically retrying the shh every few seconds until the reboot is done and ansible can ssh in again, so we know the boot worked (and we could do more tasks if we needed to). &#x20;



And there you go.  You have working ansible to play with (hopefully). &#x20;









