# OpenWRT-PVE-AP-MT7922
A guide to run OpenWRT on PVE with Access Point through m.2 wifi module MT7922


To remove the “You do not have a valid subscription for this server” popup message while logging in, run the command bellow:

    sed -Ezi.bak "s/(Ext.Msg.show\(\{\s+title: gettext\('No valid sub)/void\(\{ \/\/\1/g" /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js && systemctl restart pveproxy.service


Creating the VM

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

Setting Up the OpenWRT Disk
Select the Proxmox node name in the left navigation menu
Click Shell in the left sub-navigation
Run the following commands in the terminal

    # lookup the latest stable version number
    regex='<strong>Current Stable Release - OpenWrt ([^/]*)<\/strong>' && response=$(curl -s https://openwrt.org) && [[ $response =~ $regex ]] && stableVersion="${BASH_REMATCH[1]}"
    # download openwrt image
    wget -O openwrt.img.gz https://downloads.openwrt.org/releases/$stableVersion/targets/x86/64/openwrt-$stableVersion-x86-64-generic-ext4-combined.img.gz
    # extract the openwrt img
    gunzip ./openwrt.img.gz
    # rename the extracted img
    mv ./openwrt*.img ./openwrt.raw
    # increase the raw disk to 512 MB
    qemu-img resize -f raw ./openwrt.raw 512M
    # import the disk to the openwrt vm
    # update the vm id and storage device as needed
    # usage: qm importdisk
    qm importdisk 100 openwrt.raw local-lvm
  NB: you can update openwrt image the same way in the future, just remember to backup before hand.

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

    # set the lan ip address, use something in the same subnet as your LAN
    uci set network.lan.ipaddr='10.10.27.151'
    # restart network services
    service network restart
    # update openwrt packages
    opkg update
    # install the luci web ui
    opkg install luci
    
Open a new browser tab and navigate to http://IPofVM, http://10.10.27.151 in the example
At the login screen, enter the username root and the password set above > Click the Login button

once logged in, go to Network > Interfaces, then edit the lab inter face and set the lan ip address to the same one you set above. Don't forget to set the gateway and dns server (in advanced tab) too.

now go and download putty or any ssh terminal and follow the steps bellow:

Install wireless LAN card driver and firmware:

    opkg 

    opkg install kmod-iwlwifi iwlwifi-firmware-ax210

    opkg install kmod-mt7921e 
    cd /lib/firmware/mediatek 
    wget https://github.com/openwrt/mt76/raw/master/firmware/WIFI_MT7922_patch_mcu_1_1_hdr.bin 
    wget https://github.com/openwrt/mt76/raw/master /firmware/WIFI_RAM_CODE_MT7922_1.bin

    opkg install wpad-openssl
    reboot

after the reboot, you should be able to see wireless under network, if you can't, power circle the router (not soft reboot).

now go to wireless, enable the wireless card edit it, change country in advanced tab. then set it to ap mode, set security and password and you are set.

enjoy your private access point on openwrt as a vm on proxmox.
