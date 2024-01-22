---
description: >-
  I should have made a syslog server a long time ago, but the second best time is now.
---

# A centralized log server

So, time for a log server?

I've put this off for a while, but I'm starting to think that it would be a good idea to have a syslog server. I keep seeing people online talking about having problems with troubleshooting random lockups and reboots and not having adequate logs. And I've been thinking about doing this for some other reasons, like reducing wear on the boot SSDs for servers (not that I'm as worried about that as some people online seem to be).

### Basic plan <a href="#w1snyvjg7dpf" id="w1snyvjg7dpf"></a>

For once I am writing before doing. So here's the basic plan. I will make the syslog server with ansible role to create it. Then I will make an ansible role that can add clients, with a variable to be able to set if the machine will only log to the server, or will have local logs too.

### Where to put it? <a href="#m82defquubf" id="m82defquubf"></a>

So the first question is where should this log server live? Should it be in the trusted vlan because I wouldn't want the log files easily accessible to an attacker, or should I put it in the least trusted vlan because it would then be easier for all the other machines to talk to it to send their log to it?

I've decided on putting the server in the main network. There's only a single port that needs to be open for the log, port 514. Actually, maybe 2 since I might need to open both the TCP and UDP since some might prefer one over the other.

So, I will give the server a static IP in the trusted vlan, give it a name like syslog in the local DNS, add to the pfsense an alias for this IP, and for the less trusted VLANs add a rule to allow connections to syslog on port (TCP or UDP).

* Static IP
* DNS name
* pfsense alias
* pfsense rule in each vlan

### What the VM will look like <a href="#e0hnlxd1g76r" id="e0hnlxd1g76r"></a>

The VM itself I will deploy from my debian with LUKS template. Why with LUKS? I can't think of a good reason to not use LUKS and I am trying to make that my new default, since I can use clevis and the tang server to open that volume automatically on boot (a blog post for this will come soon). I think 16GB would be enough, so I will give it 32GB just to be sure. The general consensus seems to be that syslog-ng is easier to configure things like filtering, so I will go with that.

### Hooking things up <a href="#id-95m5ifhob0xp" id="id-95m5ifhob0xp"></a>

I will need to make an ansible role that can be added to the playbooks for other servers to configure them to send logs to syslog server (set the destination by looking up the IP address of the server from the inventory so it will still work even if DNS has a problem). I might also add a variable that can choose if local logging is on or off, if so I expect I will have it default to on.

Some things aren't managed from ansible, so will need manually configured, like pfsense, and the ubiquiti stuff. Probably should see if there is a UI for syslog config in proxmox and if so configure one machine that way to make sure I know how it wants its config files configured. Also the switches.

### Future <a href="#id-3cpt5iiz4kg" id="id-3cpt5iiz4kg"></a>

I don't have definite plans, but might someday want to try something like logstash with kibana and all that related stuff that sounds so complicated. I figure once I have the logging happening changing what is happening on that single server (even if just top copy logs to another system) shouldn't be hard. So I won't worry that far out for now.

### Execution <a href="#mdgreteydnur" id="mdgreteydnur"></a>

* Step1, pick an IP. I picked 253 from the main network because it is free.
* Step2, add it to the ansible inventory:

    syslog-server:
        ansible_host: x.x.x.253
        auto_install_normal_updates: yes
        update_auto_reboot_time: "04:50

* Step3, make a playbook:

    - name: Setup central syslog logging server
      roles:
      - luks-root-volume
      - grow-root-fs
      - common
      - automatic-updates
      - snmp
      - syslog-ng-serve

* Step 4, make the role:

    cd roles
    ansible-galaxy init syslog-ng-serve

Then write the role's tasks/main.yaml:

    - name: Install required packages
      apt:
        name: "{{ packages }}"
        state: present
      vars:
        packages:
          - syslog-ng
      notify: restart syslog-ng

    - name: Create network log data location
      file:
        dest: /data/logs
        owner: root
        group: root
        mode: "0755"
        state: directory

    - name: Copy syslog config for listening on the network
      template:
        src: logs_from_network.conf.j2
        dest: /etc/syslog-ng/conf.d/logs_from_network.conf
        owner: root
        group: root
        mode: "0644"
      notify: restart syslog-ng





And the syslog config file template:

    # TODO: Once this works, come back and change to a template that can allow
    # dividing he logs out to separate sections for the different VLANs.
    # Network source

      source s_net_all {
      # For TCP
      # I don't think that's what IP to bind to
      tcp(port(514));
      # For UDP
      udp(port(514));
      # For the IETF syslog standard?
      syslog(
        flags(no-multi-line)
        ip({{ ansible_host }})
        keep-alive(yes)
        keep_hostname(yes)
        transport(udp)
        # it's possible to include TLS it seems
        # tls()
      );
    };

    # Destination, what file(s) to send message to
    {% for network in syslog_networks %}
    destination d_{{ network.name }}_net {
      file("/data/logs/{{ network.name }}/$HOST/$YEAR/$MONTH/$DAY/$FACILITY.log" 
      owner(root) group(root) perm(0644) dir_perm(0755) create_dirs(yes));
    };

    destination d_{{ network.name }}_net_all {
      file("/data/logs/{{ network.name }}/$HOST/$YEAR/$MONTH/$DAY/all.log" 
      owner(root) group(root) perm(0644) dir_perm(0755) create_dirs(yes)); 
    };

    {% endfor %}
    # filters
    {% for network in syslog_networks %}
    filter f_{{ network.name }}_net { ( netmask({{ network.net }})) };
    {% endfor %}
    # Log directive says what source to put through what filter to decide
    # what destination
    {% for network in syslog_networks %}
    log {
      source(s_net_all);
      filter(f_{{ network.name }}_net);
      destination(d_{{ network.name }}_net);
      destination(d_{{ network.name }}_net_all);
    };

    {% endfor %}

So, now it's as easy as going to the proxmox host, deploying the VM from my template, set the IP in the cloudinit, and power it on. Once it has booted, just run the ansible playbook, and it's ready.

    ansible-playbook -i inventory.yaml --ask-vault-pass syslog_server_playbook.yaml

Then I made an asnsible playbook that can run on other machines to tell them to send logs to the syslog server. I called that syslog-send-logs. The tasks/main.yml says:

    ---
    # tasks file for syslog-send-logs
    - name: Check for rsyslog config
      stat:
        path: /etc/rsyslog.conf
        register: rsyslog_config
        
    - block:
      - name: Add config file to add remote logging
        template:
          src: 90-remote.conf.j2
          dest: /etc/rsyslog.d/90-remote.conf
          owner: root
          group: root
          mode: "0640"
      notify: restart rsyslog
    when: rsyslog_config.stat.exists == true

And the template:

    ## Add remote logging to my syslog server
    # One @ means UDP, and two @s means TCP.
    # I guess because every byte is expensive so who cares if you

    # can read it later?

    *.* @@{{ hostvars['syslog-server'].ansible_host }}:514

It was a lot easier than I expected.
