---
description: Adding on some complications
---

# The Next Level of Complication

So you have the very basics, but now lets add some complications/powers.&#x20;

## Secrets

### Secrets in files

Fist, lets consider a pretty common scenario, secrets. Ansible is all about having everything needed to set up a machine for a job in one place, but what about secrets?

For example, lets consider a reverse proxy. With ansible you can automate going from nothing to a working reverse proxy in just one or two commands. Sounds great, but this does mean that you have a copy of the SSL certificate private key. And maybe you are checking that into git. And Math help us all, maybe you are even hosting that on some public website like github!

Well, this is the sort of job encryption was made for. We can encrypt those files with a password that doesn't get checked into git, and automatically decrypt them transparently while running a playbook that needs those files.

So, for our example lets say that your certificates live in `~/ansible/roles/rproxy/files/certs.tgz`. Just run `ansible-vault encrypt roles/rproxy/files/certs.tgz` and when it asks, give it your vault password. You probably want to use the same vault password for everything (unless you want to deal with the hassle of looking up a different one for each role or something). Now if you look at that file, you will see an encoded version, like this:

```
cat roles/rproxy/files/certs.tgz 
$ANSIBLE_VAULT;1.1;AES256 
64316638386438366139396363373030366130343734396437323238323834643061326637396161 
6530656533356233316566363530653032333539383930380a373931323736666438353337303634 
...
```

### Secrets in Variables&#x20;

What about smaller secrets? Suppose you have set up a token in proxmox to allow ansible to connect to the proxmox API and clone a VM for you. This proxmox auth token key is small and rather than putting in a file, it makes more sense to put it in a variable in the inventory. We can still use ansible-vault for this.

How does it work? Easy, just run `ansible-vault encrypt_string` .  It will ask for the password, and then let you type (or paste) in the value (the API key in this example). You mark the end of the text with the keystroke ctrl-d. It will then spit out the encrypted text, which we can copy and paste into the inventory. We end up with our variable in the inventory looking like this:&#x20;

```
proxmox_token_key: !vault | 
  37643962326332363065396162303738316233663533373138623731393333666439373963633734 
  ...
```

### Using Vault Protected Secrets&#x20;

By default when you try to run a playbook that uses one of these encrypted files, it will fail because it can't decrypt the file. The easiest option, just add the `--ask-vault-pass` (or `--ask-vault-password`) to the `ansible-playbook` command. It will then ask you for the password to use to decrypt these files or variables.  This looks like `ansible-playbook --ask-vault-pass playbooks/install_updates.yml`&#x20;

You can make that the default so you don't have to type it out in all your ansible-playbook commands by settings the `DEFAULT_ASK_VAULT_PASS = True` in your ansible.cfg. The downside to this is that it will ask for the password, even if you are running a playbook that doesn't need it.

Another option is to put that password in a file outside of your normal ansible files. For example, you could put that password in `~/.ansible-password`, and then set the `vault_password_file = ~/.ansible-password` in your ansible.cfg file.



## More Complicated Playbooks With Roles

Now we can make some more complicated playbooks.  Imagine we have a few different servers that all need a mongodb server to be installed, and we don't want them to all use the same server.  We don't want to waste time writing out the steps to install and configure mongodb as part of installing these other services, ideally we just write that process once, and use it everywhere we need it. &#x20;

This is what roles are, a collection of steps to setup a machine with some configuration.  So we have some web-app that needs mongodb, and a place to save some files.  So we will make this web-app-server1 server have the roles mongodb-server, nfs-client, and our-web-app.  The playbook for that will looks like this:

```
- name: Setup seafile server
  hosts: web-app-server1
  become: yes
  roles:
    - mongodb-server
    - nfs-client
    - our-web-app
```

We could then have another playbook for another server that was also a mongodb-server, an nfs-client, but instead of our-web-app, it uses another-web-app for a different web-app.  Maximum code reuse for minimal effort! &#x20;

### Creating a Role

Ansible has a tool that helps us to build new roles, or download roles created by other people.  Here we will make a new empty role. &#x20;

```
cd roles
ansible-galaxy role create test
```

And that has made us a new empty role called test, in the `roles/test` directory. &#x20;

### The Layout of a Role

So the basic layout of this new role looks like this:

```
test/
├── defaults/
│   └── main.yml
├── files/
├── handlers/
│   └── main.yml
├── meta/
│   └── main.yml
├── README.md
├── tasks/
│   └── main.yml
├── templates/
├── tests/
│   ├── inventory
│   └── test.yml
└── vars/
    └── main.yml

```

Lets look at what each of these files and directories are for:

* defaults/main.yml \
  This is a simple place to define default values for variables this role needs.  These are almost the lowest precedence values, so setting the variable anywhere else will override the values set here.  So if you set foo here, that will be overridden by setting foo a group in the inventory, and that will be overridden by setting foo for the server being worked on in the inventory, and that would be overridden by setting foo on the command line with an argument to ansible-playbook.
* files \
  This directory is for files that the role might copy to the server being configured.  Referring back to a previous example, this is where we stored the SSL certificate files for out reverse proxy
* handlers/main.yml\
  This file holds tasks that can optionally be triggered by actions.  For example if we copy a config file for nginx we want to restart nginx for that to take effect, but if the config file was already correct on the server we don't want to restart nginx, so we do the restart as a handler.  More on this later
* meta/main.yml\
  Info about the role (author, license, etc.).  I basically never remember to do this because nothing in my work-flow uses this data. &#x20;
* README.yml\
  Documentation on the role.  This should explain to anyone else who might want to use your role how they would do that (what variables to set, what other roles it depends on, python libraries that need to be installed for this to work, etc.).
* tasks/main.yml\
  This is where most of the work happens.  The tasks needed for a role mostly go here.  When a playbook starts a role the first thing to happen is the first task from this file. &#x20;
* templates\
  This directory holds files that will be copied to the server being managed by the template command.  These are text files in the jinja2 templating language.  Variables in the template get replaced with the value of that variable.  This also can support more complicated things like if-then-else, and for loops.  This is hard to get used to, but is crazy powerful for building config files
* test/\
  This is supposed to be for tests to verify that the role works.  I never got around to figuring out how this works, to my eternal shame.
* vars/main.yml\
  This is another place to define variables, but this time near the top of the precedence.  Variables defined here override the inventory, and everything else (except maybe the command line? I think).  This is rarely used, and I've never used it. &#x20;

### Tasks

There's not much to say here.  You've already seen tasks run directly from a playbook, and they work just the same here.  Tasks are generally designed to be idempotent.  That means that a task shouldn't describe what to do, but instead describe a result.  For example if you run `echo a=5 >> /some_file`, and then run that several times, you will end up with several instances of `a=5` in the file.  Instead in an ansible task you could express the same thing as:

```
- name: Make sure a is set to 5
  ansible.builtin.lineinfile
    path: /some_file
    line: "a=5"
    regexp: "^a="
```

This uses the lineinfile module to look in the /some\_file for a line that starts with "a=", if it finds one, it makes sure that the line says "a=5", if it doesn't find one it will add "a=5".  And importantly, if it find the line is already "a=5" then it knows there's nothing that needs to be done, so it doesn't change the file.  In this way, it should be safe to run the playbook again and again. &#x20;

### Handlers

Lets extend the above example a bit more, with this example:

```
- name: Make sure a is set to 5
  ansible.builtin.lineinfile
    path: /some_file
    line: "a=5"
    regexp: "^a="
  notify: 
    - Restart nginx
```

Here we added the notify.  This means that if /some\_file was changed by this task, then after all the tasks are finished, the handler "Restart nginx" will be called.  In this way, if the file didn't need to be changed, then the restart won't need to happen either. &#x20;

This item in the handlers/main.yml file can look like this:

```
- name: Restart Nginx
  service:
    name: nginx
    enabled: yes
    state: restarted
```

That's exactly what it would look like if it were a task.  That's nice because you can easily decide to make a task into a handler just by moving it from a tasks file to a handlers file. &#x20;

You might have noticed that we triggered that handler by putting it's name on the notify list.  This means that one task could trigger lots of handlers just by listing them all.  It is also worth knowing that you could have lots of tasks that notify "Restart nginx" the same way.  What's nice about this is that if even one of those tasks actually took an action, they will trigger the handler, and if dozens of them resulted in changes on the target system, the handler will still run just once. &#x20;

## Templates (jinja2)

One of the more powerful parts of ansible is the templating system.  This can be a quick and powerful to make a config file.  Lets consider for example building an snmpd.conf config file.  We probably want most if not all of our systems to have a configuration for snmpd so we can monitor them, but some parts of this config file need to be set to a unique value for each machine.  For example, these lines set what snmp should report for a machine's location, and contact, and which IP it should listen on. &#x20;

```
sysLocation    {{ snmp_location_rack }}_{{ snmp_rack_u }}
sysContact     {{ snmp_contact }}
agentaddress   {{ ansible_host }}
```

Basically everything in the template is copied literally unless it is surrounded by double curly braces like \{{ \}} which indicate a variable context, or "mustaches" like \{% %\} which surround expressions that do things like ifs and loops, and {# #} which surrounds comments.  &#x20;

This lets us set these values for each machine by setting these variables for every machine in the inventory.  In the inventory we already have the IP for the server in ansible\_host, because have to set that for every machine, so it makes sense to reuse that here.  So here's what a machine might look like in the inventory:

```
titanium:
  ansible_host: 192.168.68.22
  snmp_location_rack: rack1
  snmp_rack_u: U20-21
  snmp_contact: Bob@it_team
```

So the config will end up looking like:

```
sysLocation    rack1_U20-21
sysContact     Bob@it_team
agentaddress   192.168.68.22
```



### Conditionals and Loops

So that's good for simple stuff, but sometimes we need more complicated configurations.  For servers that have an IPMI we want to include the fan speed reading from snmp\_ipmi in the reports. &#x20;

```
{# this section is about using IPMI sensor readings for fans #}
{% if snmp_has_ipmi is true %}
  {%- for fan in snmp_fans %}
    extend "System Fan {{ fan }}" /usr/local/bin/snmp_ipmi -f /tmp/ipmi_sensor_list -u "FAN{{ fan }}"
  {% endfor %}
{% else %}
  ## Inventory says there isn't an IPMI
  # extend "System Fan 1" /usr/local/bin/snmp_ipmi -f /tmp/ipmi_sensor_list -u "FAN1"
{% endif %}
```

We don't want an error for machines that don't have the "snmp\_has\_ipmi" set, so lets set a default value for that in our role's defaults/main.yml:

```
snmp_has_ipmi: false
```

Our server has an IPMI, and has a fan plugged into fan headers 1, 3, and A.  So in our inventory we will set:

```
  snmp_has_ipmi: true
  snmp_fans:
    - 1
    - 3
    - A
```

I am sure I don't need to explain the if - then - else - endif of this, nor the for - endfor loop.  It's probably enough to just show the resulting section of the config file:

```
    extend "System Fan 1" /usr/local/bin/snmp_ipmi -f /tmp/ipmi_sensor_list -u "FAN1"
    extend "System Fan 3" /usr/local/bin/snmp_ipmi -f /tmp/ipmi_sensor_list -u "FAN3"
    extend "System Fan A" /usr/local/bin/snmp_ipmi -f /tmp/ipmi_sensor_list -u "FANA"
```

You might have noticed that one of these \{% %\} blocks has an extra - in it.  The \{%- instead of \{% removes the extra white space before the \{% tag.  This lets us have a slightly more readable layout in the template without without it bleeding out into the results.  The - can be put on the ending tag to strip white space after a tag , like -%\} in place of the %\}.  &#x20;



