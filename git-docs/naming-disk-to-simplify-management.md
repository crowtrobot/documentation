---
description: >-
  Sometimes it can be annoying to have to interrupt what you are doing to take a
  minute to figure out which /dev/sd* is the disk you are working on.  We will
  try to simplify that process here.
---

# Naming Disk To Simplify Management

## The Problem

We've all been there; you plugged a new drive into a system and done some initial testing, and now you start typing the command for the next step only to realize that since the machine was rebooted you don't know if the drive is still /dev/sdc.  That moment of interruption where you have to take a minute to look at lsblk, or dmesg, or whatever to figure out where that drive is now isn't a huge hassle, but it also isn't always required.   Wouldn't it be nice to be able to just identify the disk in an easy way, and let udev do the work of figuring out if that's sda or sdc.  So a command like "`smartctl -a /dev/disk/by-location/front-bay-1`", or "`smartctl -a /dev/disk/by-name/backup-disk-1`" could make things just that little bit easier. &#x20;

That's a slightly contrived example, until you are working on a server with tens of drives.  It can be very handy to replaced a failed drive, and then tell zfs to use it based on where it is plugged in, like `zpool replace tank /dev/disk/by-location/front-bay-1` to start a rebuild without much effort.  But to explain what's happening, we will build in stages.  Feel free to jump to end. &#x20;

## Simply Naming A Disk

Normally if you want a name to help identify a specific file system, you would use something like LVM or partition labels, or even just use the "/dev/disk/by-id/" link for the filesystem.  But to understand a part of udev we will rely on later, lets use udev to give a disk a name. &#x20;

When a device is first identified (during boot, or a disk is plugged in, etc.) udev gathers some info about the device, and then following some rules it takes some actions like creating the appropriate device files in /dev.  We will make a new rule to automatically add a symbolic link with our disk's name, so that when the USB disk we will call "backup-drive" is plugged in udev will automatically make a link pointing to it. &#x20;

* Step 1 - We will need a unique property for the drive in question to recognize it by.  So we plug the disk in, figure out what device it is, and look at the info for that drive.  So if the disk is sda, we would run `udevadm info /dev/sda` and look through it.  Lets pick the serial number.  Here is just a section of the output:

```
E: ID_SERIAL=SanDisk_SanDisk_3.2_Gen1_A200378158F43521-0:0
E: ID_SERIAL_SHORT=A20037815
E: ID_VENDOR=SanDisk
E: ID_VENDOR_ENC=SanDisk\x20
E: ID_VENDOR_ID=0781
```

* Step 2 - Make the UDEV rule.  So lets now make the udev rule to recognize that serial number and add the link.  You can make as many of the rules as you like in a single file, no need to have tons of files.  So we can run `sudo nano /etc/udev/rules.d/81-disk-names.rules` (feel free to use whatever editor you link in place of nano) and add this line:

```
KERNEL=="sd[a-z]*", ENV{SUBSYSTEM}=="block", ACTION=="add", ENV{DEVPATH}=="/devices/pci?*", ENV{ID_USB_SERIAL_SHORT}=="A200378158F43521", SYMLINK+="/dev/disk/by-name/backup-drive1"
```

* Step 3 - Unplug the drive and plug it back in, then look in /dev/disk/by-name and you should see the link backup-drive1 pointing to some /dev/sda that is this USB disk. &#x20;

Lets break down what that rules says. &#x20;

| Test                                | Description                                                                                                                                                                                                     |
| ----------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| KERNEL=="sd\[a-z]\*"                | Only match devices the kernel gives a name that looks like "sd" followed by one or more letters.  This will leave out the partitions, we only deal with the disk.                                               |
| ENV{SUBSYSTEM}=="block"             | Only apply this rule to block devices.                                                                                                                                                                          |
| ACTION=="add"                       | Only apply this rule while the drive is being added.  This doesn't matter for this rule, but will for the rules we are building to (hint, lets not waste time running storcli when we won't need the answer).   |
| ENV{DEVPATH}=="/devices/pci?\*"     | Ignore drives that aren't connected through the PCI bus (filter out virtual disk, iscsi, nbd, rbd, loop devices, and such things).                                                                              |
| ENV{ID\_SERIAL\_SHORT}=="A20037815" | This rule only applies to devices with a serial number that is exactly "A20037815"                                                                                                                              |
| SYMLINK+="disk/by-name/backups1"    | For anything that matched the above, add this one more to the list of symlinks that udev should make                                                                                                            |

You should be careful about those equal signs.  The "==" is a match, and "+=" is basically "add this to the existing value".  If you were to use "=" instead of "+=" it would replace the already defined symlinks from previous rules.  This would mean that the other links, like the ones in  "dev/disk/by-id/" and "dev/disk/by-path/" won't get made for this device. &#x20;

## Reverse Lookups

With this it is easy enough to refer to a disk by name, like `zpool replace tank /dev/disk/by-location/raid-disk-1` but what if we want to look it up the other way around?  There's plenty of times you get an error message about /dev/sdc, and it will almost never tell you which links go with that drive.  But you can ask, with the `udevadm info` command.  You would run `udevadm info /dev/sdh` and in the output look for the lines that start with S: (I think S is for symbolic link), in the output that looks like this:

```
P: /devices/pci0000:00/0000:00:01.0/0000:01:00.0/host0/target0:0:8/0:0:8:0/block/sdh
M: sdh
J: b8:112
U: block
T: disk
D: b 8:112
N: sdh
L: 0
S: disk/by-diskseq/30
S: disk/by-uuid/2e6bc4ae-4c95-4cbd-af4b-c94c029bded6
S: disk/by-location/front_bay-3
S: disk/by-id/ata-ST12000NM0127_ZJV51TM9
S: disk/by-path/pci-0000:01:00.0-scsi-0:0:8:0
S: disk/by-location/megaraid-card-0_enclosure-front_slot-6
S: disk/by-id/wwn-0x5000c500b6349260
...
```

And there it is, sdh is front\_bay-3. &#x20;

## The Medium Complexity (no SAS Enclosures) Version

If your hardware is wired with a SATA controller port connected to each bay, then it is possible to just statically assign each port a bay name.  So, assuming you know the serial number for the drive in each bay, we can find what SATA port is connected to each bay.  So for this example /dev/sda is the drive in bay 1.  Run `udevadm info /dev/sda` and look for the DEVPATH.  For me that's `DEVPATH=/devices/pci0000:00/0000:00:1f.2/ata5/host5/target5:0:0/5:0:0:0/block/sda` .  So we can see that on the PCI device at "pci0000:00/0000:00:1f.2" there is a SATA controller, and that controller's port 5 is connected to this drive bay.  There are some extra numbers in there that I think come from mapping SATA to look like SCSI (so that 5 is copied into a few places). &#x20;

So, we can make a udev rule to recognize this (and the other bays in this example system), `emacs /etc/udev/rules.d/81-disk-names.rules` :

```
KERNEL=="sd[a-z]*", ENV{SUBSYSTEM}=="block", ENV{DEVPATH}=="/devices/pci0000:00/0000:00:1f.2/ata5.*", SYMLINK+="/dev/disk/by-location/bay1"
KERNEL=="sd[a-z]*", ENV{SUBSYSTEM}=="block", ENV{DEVPATH}=="/devices/pci0000:00/0000:00:1f.2/ata6.*", SYMLINK+="/dev/disk/by-location/bay2"
KERNEL=="sd[a-z]*", ENV{SUBSYSTEM}=="block", ENV{DEVPATH}=="/devices/pci0000:00/0000:00:1f.2/ata3.*", SYMLINK+="/dev/disk/by-location/bay3"
KERNEL=="sd[a-z]*", ENV{SUBSYSTEM}=="block", ENV{DEVPATH}=="/devices/pci0000:00/0000:00:1f.2/ata1.*", SYMLINK+="/dev/disk/by-location/bay4"
```



## Naming By Location With More Complex Controllers

So with the basics understood, we can get to the real reason here.  As noted above, I hope to avoid doing a bunch of faf to figure out what drive is the one in the front drive bay 2.  For the hardware I made this for (that will provide the examples you can build from) there are two different ways depending on how the drive is connected to the system.  We will build a system that works with both. &#x20;

This was originally make for a few different models of 2U Supermicro servers, but with the plan to be able to pretty easily extend to 4Us, 1Us, and almost anything else.  These machines have 12 SAS drive bays in front, and two 2.5" SATA bays in back.  This specific machine uses an "AVAGO MegaRAID SAS 9361-4i" as the SAS controller, but in JBOD mode because it's the 21st century, and zfs does raid better than this hardware RAID controllers. &#x20;

If we look at the devpath for one of the disks connected to this controller, we see it looks like `/devices/pci0000:00/0000:00:01.0/0000:01:00.0/host0/target0:0:5/0:0:5:0/block/sdg` .  The problem here is that none of that information actually directly and consistently maps to a drive bay.  The target 0:0:5:0 maps to what this SAS controller calls disk ID 5, but finding how that maps to a drive bay requires some more information.  So we can look at the controller's info from the storcli utility, `storcli /c0 show`

```
PD LIST :
=======

----------------------------------------------------------------------------------
EID:Slt DID State DG       Size Intf Med SED PI SeSz Model                     Sp 
----------------------------------------------------------------------------------
4:0       5 JBOD  -    3.637 TB SATA HDD N   N  512B ST4000NM0035-1V4107       U  
4:1      18 JBOD  -    5.457 TB SAS  HDD N   N  4 KB HUS726T6TAL4204           U  
4:10     21 JBOD  -    5.457 TB SAS  HDD N   N  4 KB HUS726T6TAL4204           U  
4:11     13 JBOD  -  446.625 GB SATA SSD N   N  512B Micron_5100_MTFDDAK480TCB U  
----------------------------------------------------------------------------------
```

So here we see that disk id 5 (in the DID column) is the drive in controller 0, enclosure 4, slot 0.  And for another example the disk ID 21 is in slot 10.  That disk's devpath looks like `/devices/pci0000:00/0000:00:01.0/0000:01:00.0/host0/target0:0:21/0:0:21:0/block/sdn` .

So by now you might be asking how in the world we will make such a complicated mapping work with the udev rules?  There doesn't seem to be a number we can get at easily.  What we are going to need to do is have the udev rule run a script to figure out the mapping. &#x20;

### How The Rule Will Run A Helper Script

This time we will use a single rule to call our helper script.  So we put in our /etc/udev/rules.d/81-disks-by-location.rules file, this content:

```
KERNEL=="sd[a-z]*", ENV{SUBSYSTEM}=="block", ACTION=="add", ENV{DEVPATH}=="/devices/pci?*", ENV{DEVPATH}=="/devices/pci?*", PROGRAM="/usr/local/udev-disk-locations/disk-locations.sh" SYMLINK+="%c"
```

This basically tells udev that for every block device named like /dev/sd followed by any letter or letters (but no numbers, so we aren't doing this for every partition just the drive) we need to run the helper program `/usr/local/udev-disk-locations/disk-locations.sh` and use the output as additional symlinks. &#x20;

### What The Helper Script Looks Like

This part could get pretty complicated, but I have tried to keep is as simple as possible.  First we have a simple script that will figure out which interface driver is used for this disk.  In this case, drives will either be plugged into the motherboard SATA controller which uses the AHCI driver, or the AVAGO MegaRAID SAS controller card which uses the megaraid\_sas driver.  Apologies, this used to live in a github repo, but is currently between homes while I try to find a suitable home, so for now I guess you need to copy-and-paste. &#x20;

```
#!/bin/bash
#
# GPL3
# Version: 1.0 - Working pretty well for a while, so lets call it 1.0
#
#
# This is called from udev rule, this script will try to help set
#  some links to tell you about where that specific disk is.  
# 
# For example, replace the failed drive in an md raid:
#   mdadm --manage /dev/md0 add /dev/disk/by-location/left_drive_bay
#
# Because this is run within udev, it can be tricky to debug, so
#  errors will be written to /tmp/location_error.

# Function to log an error where it is easier to find.  
location-error() {
    echo "$1" 1>2&
    LOCATION_ERROR="$LOCATION_ERROR - $1"
    echo "LOCATION_ERROR=$LOCATION_ERROR" >> /tmp/location_error
}

# Debug this script in place
#exec | tee /tmp/out.$$ 2>> /tmp/out_err.$$
#exec > /tmp/out.$$ 2>> /tmp/out_err.$$
#set -x

# there's nothing to do on remove, so lets just exit
if [[ $ACTION != add ]] ; then
    location-error "$ACTION, nothing to do"
    exit 0
fi


# Find the driver, and call the appropriate helper for that driver
if [[ -z $ID_PATH ]] ; then
    location-error "ID_PATH not set"
    exit 0
fi

# Find what directory this script is in,
# probaby /usr/local/udev-disk-locations , but it doesn't need to be
# helpers are expected to be in the same directory as this one.  
SCRIPTPATH="$( cd -- "$(dirname "$0")" >/dev/null 2>&1 ; pwd -P )"

# Directories from the DEVPATH that we should search for
#  clues about which driver is being used
Interface_path="$( echo ${DEVPATH} | cut -d / -f 1-5 )"
Interface_path2="$( echo ${DEVPATH} | cut -d / -f 1-4 )"

Interface_Driver=none
# Search for the driver in these paths
for path in "/sys/${Interface_path}/driver" "/sys/${Interface_path2}/driver" ; do
    if [[ -L $path ]] ; then
	tmp="$(basename "$( realpath "$path" )" )"
	# skip if this pointed to a pcieport driver
	if [[ $tmp == "pcieport" ]] ; then
	    continue
	fi
	Interface_Driver="$tmp"
    fi
done

# If we couldn't identify the driver, there's just nothing more to do
if [[ $Interface_Driver == none ]] ; then
    location-error "No driver found"
    exit 1
fi

# Exectute the helper for this specific driver, if it exists
driver_helper_script="${SCRIPTPATH}/location-helper-${Interface_Driver}.sh"
if [[ -e $driver_helper_script ]] ; then
    . "$driver_helper_script"
else
    location-error "No driver helper found for $Interface_Driver working on ${DEVPATH}" 
fi
```

#### AHCI

So now we need the driver-specific helpers.  Lets start with the AHCI one because it is easier to follow. &#x20;

```
#!/bin/bash
#
# GPL3
# Version: 1.0 - Working pretty well for a while, so lets call it 1.0
#
#
# The udev location  helper specific to ahci.  Effectively just look
#  up the name(s) from the ini file, and if they are found add those
#  names in /dev/disk/by-location
#
# Identify disk by SATA part, like ata5, and assign a name, like "top".
#  ata5=top
#  or
#  ata6 = bottom
#
# This gets sourced, into the disk-locations.sh script so it can call
#  functions defined there.  This shouldn't be run on its own.  
# 
# NOTE: If a computer has more than one SATA controller using the AHCI
#  driver, so that there's a controller 1 port 1, and a
#  controller 2 port 1, this would need to be rewritten to be able
#  tell these 2 ATA1s apart.  
#  

# Fail if someone tried to run this without the parent script
if [[ $( type -t location-error ) != function ]] ; then
    cat 1>&2 <<EOF
This script can't run on its own.  It requires set up of the environment
provided by the disk-locations.sh script, and should only be run by
that script.

EOF
    exit 1
fi

# should be something like ata5
ata_name="$( echo "$DEVPATH" | cut -d / -f 5 )"

# Interface_Driver should have been set by the script that called this one
# SCRIPTPATH should also be set by calling script
names_file="${SCRIPTPATH}/location_names_${Interface_Driver}.ini"
if [[ -e $names_file ]] ; then
    # For every line mentioning the ATA, like ata3
    #  loops through them, even though there should really only be one.
    #  This easily skipps entirely if the ata# isn't in the .ini file
    while read line ; do
	# cut down to anything after the =
	# and stip out any spaces
	name="$( echo "$line" | cut -d '=' -f 2 | /usr/bin/sed 's/ //g' )"

	# text to stdout is expected to be the name of a symlink
	echo "disk/by-location/${name}"
    done < <(grep -i "^${ata_name}=" "$names_file") #no-subshell this way
else
    location-error "${names} file doesn't exist"
    exit 1
fi
```

And that then depends on an .ini file,&#x20;

```
ata6=back_bay_right
ata5=back_bay_left
```

#### SAS helper

This one is much more complicated because it needs to get info from storcli.  This is also complicated by the $employer's convention that bays are numbered from left to right, bottom up, starting at 1, but the SAS back-plane numbers on this hardware count from bottom up, left to right, starting at 0. &#x20;

```
#! /bin/bash
#
# GPL3
# Version: 1.0 - Working pretty well for a while, so lets call it 1.0
#
#
# The udev location helper specific to megaraid_sas.  
#
# This gets sourced, into the disk-locations.sh script so it can call
#  functions defined there.  This shouldn't be run on its own.  
#
# Obviously not relevant for a raid, since that would be multiple drives
#  showing up as a single device.  
# 
# The SCSI ID 0:0:18:0, which is host 0, controller 0, target 18, lun 0.  
# The target 18, matches what storcli will list as the DID (drive id) of 18.
# From that we will get the enclosure number and slot number, which relate
# directly to the physical location.  The controller number appears to be 
# 0 for JBOD single disk units (which are what we are interrested in) and
# not 0 for RAIDs.  I suspect the number is the disk group, but I don't care
# to test to find out.  
#
# NOTE: I don't like it, but this script requires that you have jq installed
#  to work.  There were values I couln't get without jq when I first wrote
#  it, but I think with the official storcli instead I might be able to
#  reqrite without jq.  
#
# 

# Fail if someone tried to run this without the parent script
if [[ $( type -t location-error ) != function ]] ; then
    cat 1>&2 <<EOF
This script can't run on its own.  It requires set up of the environment
provided by the disk-locations.sh script, and should only be run by
that script.

EOF
    exit 1
fi


# Path to the storcli executable
storcli=/usr/local/bin/storcli
names_file="${SCRIPTPATH}/location_names_${Interface_Driver}.ini"

# Fail if we can't run storcli
if [[ ! -x "$storcli" ]] ; then
    locaion-error "no storcli command to run"
    exit 1
fi

# Fail if we don't have jq available
if [[ ! -x "$( which jq )" ]] ;then
    locaion-error "no jq command, which we need for parsing storcli output"
    exit 1
fi

# If we didn't get an ID_PATH, maybe not running in udev or something.
# Can't run without that.
if [[ -z $ID_PATH ]] ; then
    location-error "ID_PATH not set" 1>&2
    exit 0
fi

# Get the Host:Controller:Target:LUN and make a variable of each
HCTL="$( echo $ID_PATH | grep -io '[0-9a-f]*:[0-9a-f]*:[0-9a-f]*:[0-9a-f]*$' )"
HCTL_H="$( echo $HCTL | cut -d : -f 1 )"
HCTL_C="$( echo $HCTL | cut -d : -f 2 )"
HCTL_T="$( echo $HCTL | cut -d : -f 3 )"
HCTL_L="$( echo $HCTL | cut -d : -f 4 )"

# If the controller number isn't a 0, then this is a multi-disk device
#  (a raid), so we can't do anything.  
if [[ $HCTL_C != 0 ]] ; then
    location-error "non-zero HCTL controller number suggests this is a raid"
    exit 0
fi

# If the machine has multiple megaraid controllers, use the PCI path to
#  figure out which is the one we are looking at. 
pci="$( echo ${ID_PATH} | grep -io "^pci-[0-9a-f]*:[0-9a-f]*:[0-9a-f]*\.[0-9a-f]*" |sed 's/pci-//g' )"
pci_1="$( printf "%02x\n" "0x$( echo ${pci} | cut -d : -f 1 )" )"
pci_2="$( echo ${pci} | cut -d : -f 2 )"
pci_3="$( echo ${pci} | cut -d : -f 3 | grep -io "^[0-9a-f]*" )"
pci_4="$( printf "%02x\n" "0x$( echo ${pci} | grep -io "[0-9a-f]*$" )" )"
pci_to_compare="${pci_1}:${pci_2}:${pci_3}:${pci_4}"

# List cards from storcli in json format, use jq to select down to controller number.  
#  Loop through that list
for card in $( $storcli show J | jq '.Controllers[0]."Response Data"."System Overview"[].Ctl') ; do
    # Get controller details in json
    storcli_card_show="$( $storcli /c${card} show J )" 

    # Safety check, this isn't the right raid card if it isn't using the
    # same driver.  If that happens just skip to the next card
    stor_driver="$( echo "${storcli_card_show}" | \
    	jq -r '.Controllers[0]."Response Data"."Driver Name"' )"
    if [[ $stor_driver != $Interface_Driver ]]; then
       continue
    fi

    # Get the pci address of this controller
    stor_pci="$( echo "${storcli_card_show}" | \
        jq -r '.Controllers[0]."Response Data"."PCI Address"')"
    
    # Skip this card if the PCI location isn't the same
    if [[ $stor_pci != $pci_to_compare ]] ;then
       continue
    fi

    # get the list of enclosures
    storcli_card_enclosure_show="$( $storcli /c${card}/eall show J )"
  
    # For the drive 0:0:18:0, that 18 is what storcli calls the DID (drive ID)
    # From that ID we want to get the enclosure and slot, which looks like 4:1
    eclosure_slot_this_drive="$( echo "${storcli_card_show}" | \
      jq -r --argjson a "$HCTL_T" '.Controllers[]."Response Data"."PD LIST"[] | select(.DID==$a)."EID:Slt"' )"

    # separate enclosure and slot to spearate variables
    enclosure_id="$( echo $eclosure_slot_this_drive | cut -d : -f 1 )"
    slot_id="$( echo $eclosure_slot_this_drive | cut -d : -f 2 )"
    
    # Get the enclosure name, and the bay name from the .ini file.
    # And enclose id should look like pci-0000:03:00:0/4=front, or
    # pci-0000:03:00:0/6=jbod1-back
    if [[ -e "$names_file" ]] ; then
	enclosure_name="$( grep -i "^pci-${pci}/${enclosure_id}=" "$names_file" | head -n 1 | cut -d = -f 2 | sed 's/ //g' )"
	# if an enclosure name isn't defined, just go with enclosure-#
	if [[ $enclosure_name == "" ]] ; then
	    enclosure_name="card-${card}-enclosure-${enclosure_id}"
	fi

	# To allow one slot or bay to have multiple names defined in the
	#  names file, we will loop through all found matches.  So a drive can
	#  be jbod1-front_bay1 and jbod1-front_bottom-left if you want.
	#  You probably don't now, but that was originally how this linked
	#  both the bay# and slot# before the "link_slots=yes" was
	#  implemented.
	/usr/bin/sed 's/ //g' "$names_file" | \
	    grep -i "^${enclosure_name}.${slot_id}=" |\
	    cut -d = -f 2 | while read bay_name ; do
		# If no name is defined generate a default name
		if [[ $bay_name == "" ]] ; then
		    bay_name="bay-unknown-slot-${slot_id}"
		fi

		# std out is read as a list to be added to the symlink list
		# by the udev rule
		echo "disk/by-location/${enclosure_name}_${bay_name}"
	    done

	    # If the names file says "link_slots=yes" or true, then we will
	    #  also make a link for slot#
	link_slots="$(grep -i "^link_slots=" "$names_file" | \
	    cut -d = -f 2 | sed 's/ //g')"
	if [[ $link_slots =~ yes ]] || [[ $link_slots =~ true ]] ; then
	    echo "disk/by-location/megaraid-card-${card}_enclosure-${enclosure_name}_slot-${slot_id}"
	fi
	
    else
	# If there's no name file, we still know the card, eclosure, and slot
	#  number, so we can go with that.  
	echo "disk/by-location/megaraid-card-${card}_enclosure-${enclosure_id}_slot-${slot_id}"
    fi
done


```

And in `location_names_megaraid_sas.ini` file:

```
# Make links with the slot number too
link_slots=yes

# name the enclosure
# The raid card in pci-0000:01:00.0 has an enclosure number 4, which
# is the drive bays on the front, so pci-0000:01:00.0/4=front
pci-0000:01:00.0/4=front

# Count drive bays starting a 1, left-to-right, bottom-up.
# Slots are numbered starting at 0, botom-up, left-to-right.
# So enclosure 4 slot 0 is bay 1, and /e4/s1 us bay 4.
# front/0=bay-1 will come out like /dev/disk/by-location/front_bay-1
front/0=bay-1
front/1=bay-5
front/2=bay-9
front/3=bay-2
front/4=bay-6
front/5=bay-10
front/6=bay-3
front/7=bay-7
front/8=bay-11
front/9=bay-4
front/10=bay-8
front/11=bay-12
```

