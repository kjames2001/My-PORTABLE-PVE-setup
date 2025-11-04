# My portable server setup
A guide to setup a perfect travel nas/server on PVE with Access Point through OpenWRT on The Miroute M1Pro with m.2 wifi module MT7922 (MT7921 also works, could be even easier)

Warning: This guide is/may appear convoluted/long to some people, and may contain mistakes as I get my sources from googling and AI. Some steps are not detailed as they are easily available on the internet or specific to individual softwares. Success is not guaranteed even if you have the exact same hardware as I do.

## Features:

* portable
* can be powered by a power bank
* the *arrs
* encode media files
* immich with machine learning/face recognition/video encoding
* wifi hotspot
* openwrt firewall
* connect to hotel wifi
* connect to bluetooth peripherals
* watch media on the go, either through network or physical display
* ALL OF THE ABOVE AT THE SAME TIME!

## Hardware:
This guide is not sponsored (I wish though, lol)!

I bought the Miroute M1pro like 2 years ago, and have been playing with it ever since. And I have slowly built the software required to achive what I think is the perfect travel nas/server!

Therefore I'm finally sharing my setup, as detailed as I can be/remember.

The Miroute M1pro (N305 version) can be bought on Taobao at around $270. 

### Specs:

CPU: Intel Alder Lake i3 N305 at 8c/8t TDP 15 W*

GPU: Intel UHD Graphics*

Ram: Onboard 16gb LPDDR5 4800MHZ*

Display output: 2 HDMI 2.0 4k@60hz & 1 Type-C 4k@60hz*

Network: 2 2.5G ehternet port & 1 M.2 E key for wifi module*

Size: 75.4cm x 75.4cm x 52.5cm = 0.3L*

Power: DC 12v5a or 20v or PD 55w TYPE-C*

Bios: supports WOL/PXE and start on power up

Any spec with a * at the end is part of the reason i choosed it as my portable nas/server solusion.

When I bought the Miroute M1pro, i asked the dealer for a glass top with internal wifi antennas.

Add your own M.2 2242 nvme ssd (2TB recommanded) and a M.2 E key MT7922/MT7921 wifi module

## My use case and why it suits me:

First, its small, at 0.3 littres, its the size of a fizzy drink. Yet its still powerful, with 8 cores 8 threads, its enough for running the *arrs, encode with Tdarr, run immich with machine learning, run opencloud for syncing your non-media files and then some. Powered by a type-c port, means you don't need to take its power brick (literally) on your travels, and use any Gan charger, even a power bank. Its got both wired and wireless connections, which can connect to your home network wired for speed, and hotel wifi wirelessly for safety. Its got a type-c port for display output, which i can use with my OG nreal air glasses. Last but not least, its got a micro sd card slot, which i use to run a batocera os for gaming, could be used to extend storage capacity as well.

## Install proxmox

Install Proxmox as per usual.

Then follow the following guide.

## Proxmox dhcp setup:
To make a proxmox system truely portable, I would make use of dhcp for auto ip obtain. However, if you use the same subnet both on your home router and the to-be-installed openwrt vm, then you can skip this step.

### Network Configuration

To enable DHCP, on your server, edit /etc/network/interfaces. You should see a configuration like this (interface names may varry):
```
iface vmbr0 inet static
        address 192.168.1.157/24
        gateway 192.168.1.1
        bridge-ports enp3s0
        bridge-stp off
        bridge-fd 0
```
Modify this block and turn it into a DHCP configuration:
```
iface vmbr0 inet dhcp
        bridge-ports enp3s0
        bridge-stp off
        bridge-fd 0
```
Modify the bridge-ports accordingly, more than one port can be used here:
```
iface vmbr0 inet dhcp
        bridge-ports enp3s0 enp4s0
        bridge-stp off
        bridge-fd 0
```

### Update host names

On a Proxmox server, when updating the IP address, /etc/hosts must be updated as well. That is why just enabling DHCP can cause problems.

On Proxmox 8/9, dhclient is not installed by default, because Debian (the base) moved to systemd-networkd/systemd-resolved and recommends isc-dhcp-client only if you explicitly need it.

To fix this, there are two options:

Option 1: Install dhclient (quick fix)
```
apt update
apt install isc-dhcp-client
```
This will provide /sbin/dhclient, and ifreload -a will work with DHCP.

option 2: Use systemd-networkd for DHCP (preferred on modern Proxmox)

Instead of relying on dhclient, you can configure DHCP directly in /etc/network/interfaces like:
```
iface vmbr0 inet dhcp
        use systemd-networkd
```
This tells ifupdown2 to let systemd-networkd handle DHCP instead of looking for /sbin/dhclient.

In this portable setup with OpenWRT, Option 1 is the safest — just install isc-dhcp-client.

#### Dynamic Host Configuration

1. Create the hook file

Path: /etc/dhcp/dhclient-exit-hooks.d/update-etc-hosts
```
#!/bin/sh
#
# dhclient exit hook to update /etc/hosts when IP changes
#

# Replace these with your actual hostname and domain
HOSTNAME="pve-portable"
FQDN="pve-portable.lan"
HOST_ENTRY="${new_ip_address} ${FQDN} ${HOSTNAME}"

# Only run when DHCP lease is bound or renewed
if [ "$reason" = "BOUND" ] || [ "$reason" = "RENEW" ]; then

    # Check if the hostname already exists in /etc/hosts
    if grep -q "$FQDN" /etc/hosts; then
        # Replace the existing line with the new IP
        sed -i "s/^.*$FQDN[[:space:]].*$/${HOST_ENTRY}/" /etc/hosts
    else
        # Append the new entry if it doesn't exist
        echo "$HOST_ENTRY" >> /etc/hosts
    fi

    logger -t dhclient-hosts "Updated /etc/hosts: $HOST_ENTRY"
fi
```
Modify the HOSTNAME and FQDN values as desired.
    
2. Make it executable
```
chmod +x /etc/dhcp/dhclient-exit-hooks.d/update-etc-hosts
```
then apply the changes with:
```
ifreload -a
```
or simply a reboot.

## Remove proxmox nag

To remove the “You do not have a valid subscription for this server” popup message while logging in, run the command bellow:
```
sed -Ezi.bak "s/(Ext.Msg.show\(\{\s+title: gettext\('No valid sub)/void\(\{ \/\/\1/g" /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js && systemctl restart pveproxy.service
```
Add it to crontab if you like.

## Setting up Openwrt as a VM
### Creating the Openwrt VM

Open a web browser and navigate to the ProxMox web UI https://ProxMoxDNSorIP:8006/

Click the Create VM button at the top right

On the General tab, name the VM OpenWRT and set a VM ID (123 in this example) > click Next

On the OS tab select Do not use any media and set the Guest OS Type to Linux and Version to 5.x - 2.6 Kernel > click Next

On the System tab click Next

On the Hard Disk tab set the Disk size to 0.001 > click Next

On the CPU tab set the number of CPU cores and the Type to host > click Next

On the Memory tab set the amount of memory to 256 MiB > click Next

On the Network tab set the Model field to VirtIO (paravirtualized), Uncheck the Firewall box > click Next

On the Confirm tab review the settings and click Finish

Select the newly created OpenWRT VM from the left navigation panel

Select Hardware from the left sub-navigation menu

Click the Hard Disk to select it

Click the Detach button at the top of the main content window to detach the hard disk from the VM

Click the Unused disk to select it

Click the Remove button at the top of the main content window to permanently delete it

Click the Add button > Network Device

Set the Model field to VirtIO (paravirtualized), Uncheck the Firewall box > Click Add

### Setting Up the OpenWRT Disk

Select the Proxmox node name in the left navigation menu
Click Shell in the left sub-navigation
Run the following commands in the terminal

lookup the latest stable version number

    regex='<strong>Current Stable Release - OpenWrt ([^/]*)<\/strong>' && response=$(curl -s https://openwrt.org) && [[ $response =~ $regex ]] && stableVersion="${BASH_REMATCH[1]}"
download openwrt image

    wget -O openwrt.img.gz https://downloads.openwrt.org/releases/$stableVersion/targets/x86/64/openwrt-$stableVersion-x86-64-generic-ext4-combined.img.gz
just go to https://downloads.openwrt.org/releases/23.05.2/targets/x86/64/ and get the link there, if this doesn't work.
extract the openwrt img

    gunzip ./openwrt.img.gz
rename the extracted img

    mv ./openwrt*.img ./openwrt.raw
increase the raw disk to 512 MB

    qemu-img resize -f raw ./openwrt.raw 512M
import the disk to the openwrt vm
update the vm id and storage device as needed
usage: qm importdisk

    qm importdisk 100 openwrt.raw local-lvm

### modify vm

Once the disk import completes, select the OpenWRT VM from the left navigation menu > Hardware

Double click the Unused Disk > Click the Add button

Select Options from the left sub-navigation menu

Double click Boot Order

Check the Enabled box next to the hard disk

Drag the Hard disk up in the boot order as needed, typically below the CD-ROM device

Click OK

Double click Use tablet pointer > Uncheck the Enabled box > Click OK

Click the Start button in the top right of the screen

Click the Console link to watch the boot process

Wait for the text to stop scrolling and press Enter

Run the following command to change/set the root password

    passwd
    
Type a new root password twice to set it

Continue the configuration by running the following commands

set the lan ip address, use something in the same subnet as your LAN

    uci set network.lan.ipaddr='10.10.27.151'
restart network services

    service network restart
    
Open a new browser tab and navigate to http://IPofVM, http://10.10.27.151 in the example

At the login screen, enter the username root and the password set above > Click the Login button

once logged in, go to Network > Interfaces, then edit the lan interface and set the lan ip address to the same one you set above. Don't forget to set the gateway and dns server (in advanced tab) too.

now go and download putty or any ssh terminal and follow the steps bellow:

### Install wireless LAN card driver and firmware:

    opkg update

    opkg install kmod-iwlwifi iwlwifi-firmware-ax210

    opkg install kmod-mt7921e 
    
    cd /lib/firmware/mediatek 
    
    wget https://github.com/openwrt/mt76/raw/master/firmware/WIFI_MT7922_patch_mcu_1_1_hdr.bin 
    
    wget https://github.com/openwrt/mt76/raw/master/firmware/WIFI_RAM_CODE_MT7922_1.bin

    opkg install wpad-openssl
    
    reboot

after the reboot, you should be able to see wireless under network, if you can't, power circle the router (not soft reboot).

now go to wireless, enable the wireless card, edit it, change country in advanced tab. then set it to ap mode, set Essie name, set security and password and you are set.

### Auto restart the ap at boot:

To prevent a dead ap on startup (sometimes ap won't turn on on startup), add these lines to System > Startup > Local startup:

        uci set wireless.radio0.disabled='1' 
        uci set wireless.default_radio0.disabled='1' 
        uci commit wireless
        wifi reload
        sleep 5
        uci set wireless.radio0.disabled='0' 
        uci set wireless.default_radio0.disabled='0' 
        uci commit wireless
        wifi reload

enjoy your private access point on openwrt as a vm on proxmox.


### OpenWrt persistent repartitioning

OpenWrt has been originally developed for resource-constrained platforms. Consequently, even on x86, it doesn't have a traditional installer. Rather than install software, you copy an image onto the boot drive. That image is fairly small (about 120 MB in recent versions), so out of the box, OpenWrt has about 120 MB of total storage space regardless of the actual size of the storage device. That space can be reclaimed by repartitioning the boot drive, but that repartitioning goes the way of the dodo every time OpenWrt is upgraded.

To overcome this, we can make repartitioning persist through a minor version upgrade (current configuration will persist as well).

First, we install Attended Sysupgrade and utilities for repartitioning:

    opkg update && opkg install auc luci-app-attendedsysupgrade parted losetup resize2fs

Next, we install the repartitioning script:

    cd /root
    wget -U "" -O expand-root.sh "https://openwrt.org/_export/code/docs/guide-user/advanced/expand_root?codeblock=0"
    . ./expand-root.sh

Now we can run the repartitioning script we just installed to expand the root partition and root file system to fill the available disk space:

    sh /etc/uci-defaults/70-rootpt-resize

The device will reboot, most likely, twice. After that, the root partition and the root file system will be expanded to fill all space available to OpenWrt.

### OpenWrt sysupgrade

After all this, there are two ways to upgrade. We can type auc on the command line to run the command-line version of Attended Sysupgrade, or we can go to the management interface (System >> Attended Sysupgrade) and follow the prompts. All changes we made to our system will be preserved through the upgrade and partitioning will be maintained. The traditional sysupgrade should work just as well, except, of course, the configuration may be reset to the standard defaults.

Note that under this setup, completing the sysupgrade will require three reboots (all will be done automatically), so give your device some time to finish what it's doing.

## Setup a lxc for docker env and run your favorite docker/arrs

This part is based on your requirement/preference, so its up to you to set it up. I set it up with the docker lxc proxmox helper script.
The only thing I would share is an example of my lxc conf file:
```
#<div align='center'>
#  <a href='https%3A//Helper-Scripts.com' target='_blank' rel='noopener noreferrer'>
#    <img src='https%3A//raw.githubusercontent.com/community-scripts/ProxmoxVE/main/misc/images/logo-81x112.png' alt='Logo' style='width%3A81px;height%3A112px;'/>
#  </a>
#
#  <h2 style='font-size%3A 24px; margin%3A 20px 0;'>Docker LXC</h2>
#
#  <span style='margin%3A 0 10px;'>
#    <i class="fa fa-github fa-fw" style="color%3A #f5f5f5;"></i>
#    <a href='https%3A//github.com/community-scripts/ProxmoxVE' target='_blank' rel='noopener noreferrer' style='text-decoration%3A none; color%3A #00617f;'>GitHub</a>
#  </span>
#  <span style='margin%3A 0 10px;'>
#    <i class="fa fa-comments fa-fw" style="color%3A #f5f5f5;"></i>
#    <a href='https%3A//github.com/community-scripts/ProxmoxVE/discussions' target='_blank' rel='noopener noreferrer' style='text-decoration%3A none; color%3A #00617f;'>Discussions</a>
#  </span>
#  <span style='margin%3A 0 10px;'>
#    <i class="fa fa-exclamation-circle fa-fw" style="color%3A #f5f5f5;"></i>
#    <a href='https%3A//github.com/community-scripts/ProxmoxVE/issues' target='_blank' rel='noopener noreferrer' style='text-decoration%3A none; color%3A #00617f;'>Issues</a>
#  </span>
#</div>
arch: amd64
cores: 8
dev0: /dev/dri/card0,gid=44
dev1: /dev/dri/renderD128,gid=104
features: nesting=1
hostname: docker
memory: 14336
mp0: /mnt/docker,mp=/mnt/media
net0: name=eth0,bridge=vmbr0,hwaddr=BC:24:11:AA:8A:6A,ip=dhcp,type=veth
onboot: 0
ostype: debian
rootfs: local-lvm:vm-100-disk-0,size=200G
swap: 0
tags: community-script;docker
lxc.cgroup2.devices.allow: a
lxc.cap.drop: 
lxc.cgroup2.devices.allow: c 188:* rwm
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
```
Note the dev0: /dev/dri/card0,gid=44 and dev1: /dev/dri/renderD128,gid=104 lines, and lxc cgroup2 lines, they are for video encoding.
/mnt/docker is a dir on the host where the sd card/external ssd is mounted for media storage.
Network uses default, no firewall.

## Setup a script/service to automatically start openwrt vm if no ethernet cable is detected on enp3s0 socket

When we are at home, we do not need the openwrt vm, as proxmox optains its IP through your home router/firewall. So when you plug in an ethernet cable in enp3s0 (left socket on the Miroute M1pro), we do not want openwrt vm to start. Whereas while we are travelling, when we power on the host machine and want to connect to hotel wifi, then we would want openwrt to start.

This script automates that exact scenario. 
Create the script:
```
nano /root/scripts/checkip.sh
```
with these content:
```
#!/bin/bash
# Log file
LOGFILE="/var/log/checkip.log"
# Redirect all output (stdout & stderr) to log file with timestamps
exec > >(while read line; do echo "$(date '+%F %T') - $line"; done >> "$LOGFILE") 2>&1
echo "=== Script started ==="
# Quick check with minimal delay
check_nic() {
    # Wait briefly for carrier file to exist
    for i in {1..3}; do
        if [ -f /sys/class/net/enp3s0/carrier ]; then
            carrier=$(cat /sys/class/net/enp3s0/carrier 2>/dev/null)
            echo "Check attempt $i: enp3s0 carrier = $carrier"
            [ "$carrier" = "1" ] && return 1  # Cable detected, return false
            sleep 0.5
        else
            sleep 0.5
        fi
    done
    return 0  # No carrier detected after retries
}

if check_nic; then
    echo "NIC enp3s0 is DOWN (no cable). Starting OpenWRT + AdGuard..."

    # Start OpenWRT VM if not running
    if ! qm status 101 | grep -Eq "running|starting|stopping"; then
        echo "Starting OpenWRT VM 101..."
        /usr/sbin/qm start 101
    else
        echo "OpenWRT VM 101 already running."
    fi

    # Start AdGuard LXC if not running
    if ! pct status 102 | grep -Eq "running|starting|stopping"; then
        echo "Starting AdGuard LXC 102..."
        /usr/sbin/pct start 102
    else
        echo "AdGuard LXC 102 already running."
    fi

    # Give OpenWRT a head start before DHCP renew
    sleep 5

    # Renew DHCP on Proxmox
    echo "Renewing DHCP lease on vmbr0..."
    /usr/sbin/dhclient -v -r vmbr0 2>/dev/null
    /usr/sbin/dhclient -v vmbr0
else
    echo "NIC enp3s0 is UP (cable detected). Skipping OpenWRT start."
#    /usr/sbin/dhclient -v vmbr0
fi

GPU_DEV="/dev/dri/$(basename /sys/class/drm/renderD128/device/drm/card*)"
echo "Detected iGPU at $GPU_DEV"
sed -i "s|/dev/dri/card[0-9]*|$GPU_DEV|g" /etc/pve/lxc/100.conf
echo "Updated /etc/pve/lxc/100.conf to use $GPU_DEV"
sleep 1  # let pmxcfs commit changes

# Start LXC 100
if ! pct status 100 | grep -Eq "running|starting|stopping"; then
    echo "Starting LXC 100..."
    /usr/sbin/pct start 100
else
    echo "LXC 100 already running."
fi
echo "Started LXC 100"

if ! pct status 100 | grep -q "running"; then
    echo "ERROR: LXC 100 failed to start!"
fi
```

Give it permission to excute:
```
chmod +x /root/scripts/checkip.sh
```
Then create a service file:
```
nano /etc/systemd/system/checkip.service
```
with these content:
```
[Unit]
Description=Check IP and start OpenWRT + AdGuard if needed
After=local-fs.target network-pre.target pve-cluster.service pvedaemon.service
Requires=pve-cluster.service pvedaemon.service

[Service]
Type=oneshot
ExecStart=/root/scripts/checkip.sh
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
```
Enable and start the service:
```
systemctl daemon-reload && systemctl enable checkip.service && systemctl start checkip.service
```

Note: With this setup, if an ethernet cable is plugged into enp4s0, openwrt will still start and utilize that port for internet access. This is to be used in untrustworthy environments like hotels that provide a lan port into their network.

## Setup a Kodi lxc for media play back with display output
I set this up because I wanted to make use of the display output of the iGPU, while simutaneously utilize its encoding/decoding part in another lxc.
I have an OG nreal air for a travel display, but I'm sure any XR/AR glasses will do. Hell, even a small portable display will work well.

For manual setup, follow this guide: https://blog.konpat.me/dev/2019/03/11/setting-up-lxc-for-intel-gpu-proxmox.html

For automatic setup, run this command in the Proxmox VE Shell:
```
bash -c "$(wget -qLO - https://raw.githubusercontent.com/kjames2001/proxmoxHelper/main/ct/kodi-v1.sh)"
```
I only changed unprivileged lxc to privileged lxc for better compatibility, every other setting was default.

Check your lxc conf file, it should look something like this:
```
## kodi LXC
arch: amd64
cores: 4
dev0: /dev/fuse
features: nesting=1
hostname: kodi
memory: 4096
mp0: /mnt/docker,mp=/mnt/docker
net0: name=eth0,bridge=vmbr0,hwaddr=BC:24:11:0D:B2:88,ip=dhcp,type=veth
onboot: 0
ostype: ubuntu
rootfs: local-lvm:vm-103-disk-0,size=20G
swap: 512
lxc.mount.entry: /dev/fb0 dev/fb0 none bind,optional,create=file
lxc.mount.entry: /dev/dri/renderD128 dev/renderD128 none bind,optional,create=file
lxc.mount.entry: /dev/tty7 dev/tty7 none bind,optional,create=file
lxc.cgroup2.devices.allow: c 226:0 rwm
lxc.cgroup2.devices.allow: c 226:128 rwm
lxc.cgroup2.devices.allow: c 29:0 rwm
lxc.mount.entry: /dev/dri dev/dri none bind,optional,create=dir
lxc.cgroup2.devices.allow: c 4:7 rwm
lxc.cgroup2.devices.allow: c 13:* rwm
lxc.mount.entry: /dev/input dev/input none bind,optional,create=dir
lxc.cgroup2.devices.allow: c 116:* rwm
lxc.mount.entry: /dev/snd dev/snd none bind,optional,create=dir
lxc.cgroup2.devices.allow: a
lxc.cap.drop: 
lxc.cgroup2.devices.allow: c 188:* rwm
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
```
Note the /dev/fuse passthrough, its important for input detection. Add it if its not there.
/mnt/docker is also mounted here, so that kodi has direct access to media files.

### Setup audio

To get audio output through the hdmi port, run the following commands in the lxc console:
```
aplay -l
```

this should give you an output like this:
```
**** List of PLAYBACK Hardware Devices ****
card 1: Device [USB Audio Device], device 0: USB Audio [USB Audio]
Subdevices: 1/1
Subdevice #0: subdevice #0
card 2: Generic_1 [HD-Audio Generic], device 3: HDMI 0 [LG TV]
Subdevices: 1/1
Subdevice #0: subdevice #0
card 2: Generic_1 [HD-Audio Generic], device 7: HDMI 1 [HDMI 1]
Subdevices: 1/1
Subdevice #0: subdevice #0
card 2: Generic_1 [HD-Audio Generic], device 8: HDMI 2 [HDMI 2]
Subdevices: 1/1
Subdevice #0: subdevice #0
card 2: Generic_1 [HD-Audio Generic], device 9: HDMI 3 [HDMI 3]
Subdevices: 1/1
Subdevice #0: subdevice #0
```
test for audio output with this command:
```
aplay -c1 -D plughw:CARD=Generic_1,DEV=3 /usr/share/sounds/alsa/Front_Center.wav
and/or
aplay -D plughw:2,3 /usr/share/sounds/alsa/Front_Center.wav
```
then edit /etc/asound.conf accordingly:
```
defaults.pcm.card 2
defaults.pcm.device 3
defaults.ctl.card 1
```

### Optional: install xfce desktop env on the lxc and give user kodi permission to shutdown the lxc
I installed xfce desktop environment on top of the existing kodi, thought i would explorer more options like running a web browser instead of just playing media with a display connected.

I went through the trouble of creating a script to automate the procedure. Run the following command in the lxc console to do this automatically:
```
bash -c "$(wget -qLO - https://raw.githubusercontent.com/kjames2001/proxmoxHelper/main/setup/xfce-install.sh)"
```

This script also adds permission to the user kodi to shutdown the lxc. This will come in handy in the next optional part.

### Optional: Automate the start up and shutdown of the lxc

I then went further to write a script that detects when a display is been plugged into the host machine, and start/stop the lxc when the display is plugged/unplugged.
create script:
```
nano /root/scripts/kodi.sh
```
with these content:
```
#!/bin/bash
# Starts LXC when a display is connected, stops when none are connected
# Won't restart if container was stopped manually
KODI_LXC="103"
FLAG_FILE="/tmp/kodi-hotplug-auto-started"
MANUAL_STOP_FLAG="/tmp/kodi-hotplug-manual-stop"
STATE_FILE="/tmp/kodi-hotplug-prev-state"
CHECK_INTERVAL=5  # seconds

while true; do
    # Initialize counter
    DISPLAY_CONNECTED=0
    
    # Loop over all DRM connectors
    for STATUS_FILE in /sys/class/drm/*/status; do
        if [ -f "$STATUS_FILE" ]; then
            CONTENT=$(cat "$STATUS_FILE")
            if [ "$CONTENT" = "connected" ]; then
                DISPLAY_CONNECTED=$((DISPLAY_CONNECTED + 1))
            fi
        fi
    done
    
    # Check if LXC is running
    IS_RUNNING=$(pct status "$KODI_LXC" 2>/dev/null | grep -q "status: running" && echo "true" || echo "false")
    
    # Read previous state if exists
    PREV_RUNNING=""
    if [ -f "$STATE_FILE" ]; then
        PREV_RUNNING=$(cat "$STATE_FILE")
    fi
    
    # Detect manual stop: was running, now stopped, display still connected
    if [ "$PREV_RUNNING" = "true" ] && [ "$IS_RUNNING" = "false" ] && [ "$DISPLAY_CONNECTED" -gt 0 ]; then
        echo "$(date) - Manual stop detected"
        touch "$MANUAL_STOP_FLAG"
        rm -f "$FLAG_FILE"
    fi
    
    if [ "$DISPLAY_CONNECTED" -gt 0 ]; then
        # Display connected
        
        if [ "$IS_RUNNING" = "false" ]; then
            # Container is stopped
            if [ -f "$MANUAL_STOP_FLAG" ]; then
                # Manual stop flag exists, don't start
                : # Do nothing silently
            else
                # No manual stop flag, safe to start
                echo "$(date) - Display detected, starting Kodi LXC..."
                /usr/sbin/pct start $KODI_LXC
                touch "$FLAG_FILE"
            fi
        else
            # Container is running
            if [ ! -f "$FLAG_FILE" ]; then
                # Running but we didn't start it - adopt it
                touch "$FLAG_FILE"
            fi
            # Clear manual stop flag since it's running
            rm -f "$MANUAL_STOP_FLAG"
        fi
        
    else
        # No display connected
        
        # If container is running and we're managing it, stop it
        if [ "$IS_RUNNING" = "true" ] && [ -f "$FLAG_FILE" ]; then
            echo "$(date) - No display detected, stopping Kodi LXC..."
            /usr/sbin/pct stop $KODI_LXC
        fi
        
        # Clean up all flags when no display
        rm -f "$FLAG_FILE"
        rm -f "$MANUAL_STOP_FLAG"
        rm -f "$STATE_FILE"
    fi
    
    # Save current state for next iteration
    echo "$IS_RUNNING" > "$STATE_FILE"
    
    sleep $CHECK_INTERVAL
done
```
Give it permission to excute:
```
chmod +x /root/scripts/kodi.sh
```
Add a system service to call the script at the right time, by creating a file: 
```
nano /etc/systemd/system/kodi.service
```
With the following content:
```
[Unit]
Description=Hotplug manager for Kodi lxc
After=local-fs.target network-pre.target pve-cluster.service pvedaemon.service multi->
Wants=multi-user.target

[Service]
Type=simple
ExecStart=/root/scripts/kodi.sh
Restart=always
RestartSec=10
User=root

[Install]
WantedBy=multi-user.target
```
Enable and start the service:
```
systemctl daemon-reload && systemctl enable kodi.service && systemctl start kodi.service
```
This service/script runs every 5 secs to detect for plug/unplug of display and start/stop the lxc accordingly, but will do nothing if the lxc is started/stoped before/after the plugging event. 
That means if I plug in the display after i manually started the lxc, it will do nothing. And if I manually shutdown the lxc after i plugged in the display, it will also do nothing.
The shutdown permission add to user kodi above, was so that if you need to access proxmox console via the display for any reason, then you could shutdown the lxc and it will not automatically start again while your display is plugged in.

### Optional: Bluetooth
Since /dev/fuse is passed through to the lxc, any input should also be captured by the lxc. 
Therefore if a bluetooth mouse/keyboard/gamepad/controller is paired with the host, we should be able to use it in kodi/xfce when the lxc is started.
A quick google search gave me the commands I needed to pair with the bluetooth gamepad I had to control kodi wirelessly.
Install bluez in the pve shell:
```
apt install bluez
```
Then follow this guide summarized by google gemini:
1. Open bluetoothctl: Launch the command-line tool by typing bluetoothctl in your terminal.
2. Power on the adapter: If your Bluetooth controller is off, type power on.
3. Register the pairing agent: Type agent on to enable pairing. If it says it's already registered, you can proceed.
4. Scan for devices: Type scan on to start searching for nearby Bluetooth devices. Wait for your device's MAC address to appear in the list.
5. Pair the device: Once the device is visible, type pair <MAC_ADDRESS>, replacing <MAC_ADDRESS> with the device's address. You may be prompted for a PIN or confirmation.
6. Trust the device: If you plan to reconnect later without re-pairing, type trust <MAC_ADDRESS> to automatically allow connections in the future.
7. Connect to the device: To establish a connection, type connect <MAC_ADDRESS>.
8. Exit bluetoothctl: Type exit to leave the program

Make sure your bluetooth device and a mouse are connected before you plug in the display/start the lxc.
After you are in kodi, press the home button or any button really on your controller/gamepad, kodi should immediately recognize it and allow you to configure it in settings.

### TO-DO: Gaming, web surfing, office work, etc.
Since xfce is installed, I think the possibilities are endless. You could be playing games (kodi has game plugin called Mame), or install firefox, libre office, etc. I'm sure there are things I haven't mentioned or even thought of. 
Let me know in issues if you have better ideas and how to achieve them in this little portable setup.
Thank you for going through my setup with me all this way. Have fun travelling with this setup.
