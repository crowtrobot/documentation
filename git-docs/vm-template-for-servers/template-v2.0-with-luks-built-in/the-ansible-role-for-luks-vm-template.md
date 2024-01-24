# The Ansible Role For LUKS VM template

This is the role that does the actions I described previously.  I decided to break it out to its own doc to make it easier for someone to skip if they don't care about the asnible part.  

# tasks/main.yml

First we will install programs we will need
    
    - name: Update apt, and install basic tools needed for clevis
      apt:
        name: "{{ packages }}"
        state: present
      vars:
        packages:
          - clevis
          - clevis-initramfs
          - clevis-luks
          - clevis-tpm2
          - clevis-systemd

Then figure out the device that / is mounted from.  So root\_dev can be used as a variable later to refer to this device.  This will look like /dev/mapper/root-crypt

    - name: mount device name
      ansible.builtin.set_fact:
        root_dev: "{{ ansible_mounts | json_query('[?mount == `/`].device') | json_query('[0]') | string }}"

Set crypt\_vol\_name varialbe to just the volume name part (the root-crypt part of /dev/mapper/root-crypt).  

    - name: find the root device
      shell: echo "{{ root_dev }}" | grep  -o '[^/]*$'
      register: crypt_vol_name

If the / is not mounted from something in /dev/mapper, then you are probably running this role against a machine that doesn't have the root filesystem encyrpted.  If that's the case, we can just skip the rest of this.  I'm sure there's a better way to exit early, but I haven't found it yet.  

    - name: Skip this role on this host if root isn't mounted from /dev/mapper dev
      ansible.builtin.set_fact:
        using_mapper: no
      when: "not 'dev/mapper' in ansible_mounts|json_query('[?mount == `/`].device')|string"

    - name: Coninue this role on this host if root is mounted from /dev/mapper dev
      ansible.builtin.set_fact:
        using_mapper: yes
      when: "'dev/mapper' in ansible_mounts|json_query('[?mount == `/`].device')|string"

If the root is mounted from a device in /dev/mapper, go on to do the rest of this.  Otherwise just skip to the end of this file.  

    - block:
      - name: get crypt volume parent partition
        shell: dmsetup deps {{ root_dev }} -o devname | sed 's/.*(\(.*\))/\1/g'
        register: crypt_partition

      - name: Does the initial_password file still exist?
        stat:
          path: /boot/initial_password
        register: file_data

      - name: Run the clevis_setup test if initial password still exists
        include: clevis_setup.yml
        when: file_data.stat.exists
      when: "using_mapper == true"



# tasks/clevis_setup.yml

Find out what device contains the LUKS vlume (this should look like /dev/sda3).  

    - name: get crypt volume parent partition
      shell: dmsetup deps {{ root_dev }} -o devname | sed 's/.*(\(.*\))/\1/g'
      register: crypt_partition

Verify that the fallback password is set.  This should be a variable set in the inventory.  It should be encrypted with ansible valut.  If the variable is not set, or wasn't read because we didn't include the --ask-vault-pass switch, end running the playbook.  

    # Warn, if the fallback password is still the default.  It should be a 
	# vault encrypted variable, so in addition to  testing if it wasn't set, 
	# this will cause an early fail if the vault password wasn't provided, 
	# saving us from doing half the work and then failing.

    - block:
      - name: check if fallback password is the default
        debug:
          msg: |
            Warning, using the default fallback password.
            That's fine for testing, but don't use it for actual
            important machines.

      - meta: end_host
      when: luks_fallback_password == "password"

Do the re-encrypt so this VM doesn't use the same key as the template or any other VM.  

    - name: Re-encrypt to get new master key (will take a few minutes)
      shell: cryptsetup reencrypt /dev/{{ crypt_partition.stdout }} -d /boot/initial_password 

I put my tang servers on the IP ending in 254 in each of my subnets.  This sets that IP as the variable tang_ip

    - name: figure out tang server
      shell: echo "{{ ansible_default_ipv4.network }}" | sed 's/0$/254/g'
      register: tang_ip

Add tang server as a clevis pin that can unlock LUKS.  

    - name: Add tang key
      shell: timeout 60 clevis luks bind -d /dev/{{ crypt_partition.stdout }} -s8 -k /boot/initial_password -y tang '{"url":"http://{{ tang_ip.stdout }}"}'
      # don't try to use tang server to unlock its own luks
      when: tang_ip.stdout != ansible_host
      
Add the fallback password to LUKS.  
	  
    - name: Change password
      community.crypto.luks_device:
        device: "/dev/{{ crypt_partition.stdout }}"
        keyfile: /boot/initial_password
        new_passphrase: "{{ luks_fallback_password }}"
        pbkdf:
          iteration_time: 8

Remove the old initial_passsword file as a password from LUKS, and reote the file.  

    - name: Remove old initial password
      community.crypto.luks_device:
        device: "/dev/{{ crypt_partition.stdout }}"
        remove_keyfile: /boot/initial_password
      
    - name: Remove the initial password file
      ansible.builtin.file:
        path: /boot/initial_password
        state: absent

Change crypttab to not expect to find the password in a file anymore.  

    - name: Modify crypttab
      ansible.builtin.lineinfile:
        path: /etc/crypttab
        backrefs: yes
        line: '\1 none \2'
        regexp: '^({{ crypt_vol_name.stdout }}.*)/boot/initial_password(.*)$'

Change the initrd config to not include the initial-password file anymore, and then rebuild the initrd.

    - name: No need to include the initial-password in initramfs anymore
      lineinfile:
        state: absent
        path: /etc/cryptsetup-initramfs/conf-hook
        regexp: "KEYFILE_PATTERN=/boot/initial_password"
        
    - name: update initramfs
      shell: update-initramfs -u -k all ; sync

    - name: update grub
      shell: update-grub2

Reboot so if there were any problems we find out before wasting any more time on the setup.  

    - name: Reboot to verify this all worked
      reboot:


