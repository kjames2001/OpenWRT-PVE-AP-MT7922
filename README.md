# OpenWRT-PVE-AP-MT7922
A guide to run OpenWRT on PVE with Access Point through m.2 wifi module MT7922

## Proxmox dhcp setup:

### Network Configuration

To enable DHCP, on your server, edit /etc/network/interfaces. You should see a configuration like this (interface names may varry):
```
iface vmbr0 inet static
        address 192.168.1.157/24
        gateway 192.168.1.1
        bridge-ports enp5s0
        bridge-stp off
        bridge-fd 0
```
Modify this block and turn it into a DHCP configuration:
```
iface vmbr0 inet dhcp
        bridge-ports enp5s0
        bridge-stp off
        bridge-fd 0
```
Modify the bridge-ports accordingly

On Proxmox 8/9, dhclient is not installed by default, because Debian (the base) moved to systemd-networkd/systemd-resolved and recommends isc-dhcp-client only if you explicitly need it.

To fix this, there are two options:

Option 1: Install dhclient (quick fix)
```
apt update
apt install isc-dhcp-client
```
This will provide /sbin/dhclient, and ifreload -a will work with DHCP.

ption 2: Use systemd-networkd for DHCP (preferred on modern Proxmox)

Instead of relying on dhclient, you can configure DHCP directly in /etc/network/interfaces like:
```
iface vmbr0 inet dhcp
        use systemd-networkd
```
This tells ifupdown2 to let systemd-networkd handle DHCP instead of looking for /sbin/dhclient.

In this portable setup with OpenWRT, Option 1 is the safest — just install isc-dhcp-client.

### Dynamic Host Configuration

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

## Start openwrt at boot and renew dhcp lease if no IP is obtained

To check for internet access @reboot, and renew dhcp lease from openwrt vm if no ethernet cable is plugged in:

1. Script → /root/scripts/checkip.sh
```
#!/bin/bash

# Log file
LOGFILE="/var/log/checkip.log"

# Redirect all output (stdout & stderr) to log file with timestamps
exec > >(while read line; do echo "$(date '+%F %T') - $line"; done >> "$LOGFILE") 2>&1

echo "=== Script started ==="

# Function to check if IP is obtained
check_ip() {
    ip=$(ip addr show | grep inet | grep -vE "127\.0\.0\.1|fe80::|::1" | awk '{print $2}' | cut -d'/' -f1)
    [ -n "$ip" ]  # returns 0 if not empty, 1 if empty
}

# Start timer
start=$(date +%s)

# Wait loop for 5 seconds to see if host already has IP
while [ $(($(date +%s) - $start)) -lt 5 ]; do
    if check_ip; then
        echo "Host already has IP: $ip"
        echo "=== Script finished ==="
        exit 0
    fi
    sleep 1
done

echo "No IP detected, starting OpenWRT (VM 101) and AdGuard (LXC 102)..."

# Start OpenWRT VM if not running
if ! qm status 101 | grep -q "running"; then
    echo "Starting OpenWRT VM 101..."
    /usr/sbin/qm start 101 &
else
    echo "OpenWRT VM 101 already running."
fi

# Start AdGuard LXC if not running
if ! pct status 102 | grep -q "running"; then
    echo "Starting AdGuard LXC 102..."
    /usr/sbin/pct start 102 &
else
    echo "AdGuard LXC 102 already running."
fi

sleep 20

# Renew DHCP on Proxmox
echo "Renewing DHCP lease on vmbr0..."
/usr/sbin/dhclient -v -r vmbr0 2>/dev/null
/usr/sbin/dhclient -v vmbr0

echo "=== Script finished ==="
```

Make it executable:
```
chmod +x /root/script/checkip.sh
```
2. Systemd Service → /etc/systemd/system/checkip.service
```
[Unit]
Description=Check IP and start OpenWRT + AdGuard if needed
After=pve-cluster.service pvedaemon.service
Requires=pve-cluster.service pvedaemon.service

[Service]
Type=oneshot
ExecStart=/root/checkip.sh
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
```

3. Enable & Start
```
systemctl daemon-reload
systemctl enable checkip.service
systemctl start checkip.service
```
Logs:
```
journalctl -t checkip -f
```

## Remove proxmox nag

To remove the “You do not have a valid subscription for this server” popup message while logging in, run the command bellow:

    sed -Ezi.bak "s/(Ext.Msg.show\(\{\s+title: gettext\('No valid sub)/void\(\{ \/\/\1/g" /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js && systemctl restart pveproxy.service

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
  NB: you can update openwrt image the same way in the future, just remember to backup before hand.

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

once logged in, go to Network > Interfaces, then edit the lab inter face and set the lan ip address to the same one you set above. Don't forget to set the gateway and dns server (in advanced tab) too.

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

