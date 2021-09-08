# udm-blueTV

A tutorial on how to get Swisscom blue TV working with the UDM and Ubiquiti Switches. Original forked from [peacey/udm-telus](https://github.com/peacey/udm-telus) an adpoted for Swisscom blue TV

### TABLE OF CONTENTS

1. [INTRODUCTION](#introduction)
2. [PREREQUISITES](#prerequisites)
3. [CREATE A VLAN FOR YOUR TV TRAFFIC](#create-vlan-for-tv-traffic)
4. [ADD FIREWALL RULES TO ALLOW TV TRAFFIC INTO YOUR NETWORK](#add-firewall-rules)
5. [INSTALL AND CONFIGURE IGMPPROXY](#install-and-configure-igmpproxy)
6. [CONFIGURE IGMPPROXY TO RUN ON BOOT](#running-igmpproxy-on-boot)
7. [CREDITS](#credits)

### INTRODUCTION 

Swisscom blue TV uses multicast to deliver their TV streams to the Swisscom blue TV boxes. The UDM and UDM-PRO can do multicast routing since version 1.11.0 and above, but it has to be configured manually through the command line. If not configured, TV channels will only play for 10 seconds before stopping if you are watching linear TV, replay will always work. 

This tutorial will show you how to install and configure igmpproxy for Swisscom blue TV, and how to connect your blue TV Box to the UDM-PRO.
Following this tutorial successfully will allow you to watch Swisscom blue TV without interruption with your Ubiquiti equipment. 

Most of this tutorial uses the new UI, so it is recommended to follow this tutorial with the new UI unless you know where the options are in the old UI. The UDM Pro and UDM Base are both supported.

### PREREQUISITES 

Make sure you check the following prerequisites before trying the other steps:

The kernel on your UniFi device must support multicast routing in order to support IPTV:

   * UniFi Dream Machine (Pro): Multicast routing is supported natively in the stock kernel since firmware version 1.11 or above.
   * UniFi Dream Machine Pro SE: You need Early Access firmware 2.3.7+ or above for multicast routing support.
   * UniFi Dream Router: Multicast routing is supported by the default firmware.
   * The switches in-between the IPTV decoder and the UniFi device should have IGMP snooping enabled. They do not need to be from Ubiquiti necessarily.
   * The FTTP NTU (or any other type of modem) of your ISP must be connected to one of the WAN ports of your UniFi device - WAN2(SFP+,eth9) or WAN(RJ45,eth8) on an UDM-PRO oder WAN(RJ45,eth4).
   * SSH enabled on your UDM

### CREATE VLAN FOR TV TRAFFIC

This step is optional but recommended so that Swisscom blue TV traffic does not flood your other networks. Creating a separate VLAN will allow us to isolate IPTV traffic to just this network. 

1. In your UDM-PRO network controller, go to Settings -> Networks -> Add New Network. Set *Name* to IPTV. Change the following options under Advanced.
    * *VLAN ID:* 70
    * Disable *Auto Scale Network.*
    * Set *Gateway IP/Subnet* to something unique like 192.168.70.1/24. Click *Auto-Configure* under DHCP Range or adjust for your needs. 
    * Change *IPv6 Interface Type* to Prefix Delegation to enable IPv6, or set it to None to disable IPv6. IPv6 is not required for Swisscom blueTV to work, so it is optional to enable it.
    * If you do opt to use IPv6, make sure you configured your WAN's IPv6 settings first under Settings -> Internet -> Click your WAN -> Advanced -> Set IPv6 Connection* to DHCPv6 and *Prefix Delegation Size* to 60.

2. Hit *Add Network* to save. 

3. Note if you are using the SFP+ port for your Internet (WAN2) and enabled IPv6, you need to change the *IPv6 Prefix Delegation Interface* option in the old UI to WAN2. This option is not available yet in the new UI. To change this option, do the following:
	* Switch to the Old UI by disabling Settings -> System -> New User Interface.
	* Go to Settings -> Networks -> Edit IPTV Network -> Configure IPv6 Network, and set *IPv6 Prefix Delegation Interface* to WAN2.

4. Put your Swisscom TV Box on the new VLAN network by changing the port profile to IPTV on the switch ports connected to your Swisscom TV box.
	* If your Swisscom TV box is connected directly to the UDM: Go to Unifi Devices -> Click the UDM -> Settings -> Click the port the box is connected to -> Change Port Profile to IPTV -> Apply Changes. 
	* If your Swisscom TV box is connected to a downstream Ubiquiti switch: Go to Unifi Devices -> Click the switch the box is connected to -> Settings -> Click the port the box is connected to -> Change Port Profile to IPTV -> Apply Changes. 
	* Unplug and replug the Swisscom TV box so it obtains an IP on the new VLAN. 

### ADD FIREWALL RULES

1. Add the IPTV Multicast Rule to allow IPTV multicast traffic into your WAN. 
    * On your UDM, go to Settings -> Traffic & Security -> Global Threat Management -> Firewall -> Create New Rule. 
    * Set *Type:* **Internet In**. *Description:* Allow IPTV Multicast. *IPv4 Protocol:* **UDP**.
    * Under Source - *Source Type:* Address/Port Group. Click IPv4 Address Group -> Create New Group.
    * Change group's name to blueTV. Choose type IPv4 Address/Subnet. Click Add Address and add the following 3 addresses (these are the addresses blueTV uses for IPTV):
        * 195.186.0.0/16
        * 213.3.72.0/24
        * 224.0.0.0/4
    * Click Create New Group to create the group and make sure "blueTV" is selected under IPv4 Address Group.
    * Under Destination - *Destination Type:* Address/Port Group. Click IPv4 Address Group -> Create New Group. 
       * Change group's name to IGMP. Choose type IPv4 Address/Subnet. Click Add Address and add the following address:
         * 224.0.0.0/4
       * Click Create New Group to create it and make sure "IGMP" is selected under IPv4 Address Group.
    * Hit Apply Changes to save the rule. 

2. Add the IGMP Rule to allow IGMP traffic to travel across your WAN.
    * On your UDM, go to Settings -> Traffic & Security -> Global Threat Management -> Firewall -> Create New Rule. 
    * Set *Type:* **Internet Local**. *Description:* Allow IGMP Traffic. *IPv4 Protocol:* **IGMP**.
    * Under Advanced, enable all four match options: Match State New, Match State Established, Match State Invalid, Match State Related. 
    * Hit Apply Changes to save the rule.

### INSTALL AND CONFIGURE IGMPPROXY

**Your UDM must be updated to at least version 1.11.0 or above to use igmpproxy.**

1. SSH into your UDM. Replace 192.168.1.1 with your UDM's IP. 

    ```sh
    ssh root@192.168.1.1
    ```
  
2. Download igmpproxy and configuration file.

    ```sh
    mkdir /mnt/data/igmpproxy
    ```
    
    ```sh
    cd /mnt/data/igmpproxy
    ```
    
    ```sh
    curl -Lo igmpproxy https://raw.githubusercontent.com/bprskalo/udm-blueTV/main/igmpproxy
    ```
    
    ```sh
    curl -Lo igmpproxy.conf https://raw.githubusercontent.com/bprskalo/udm-blueTV/main/igmpproxy.conf
    ```
    
    ```sh
    chmod +x igmpproxy
    ```
    
* Note the igmpproxy binary included in this repository was extracted directly from the [debian arm64 package](https://packages.debian.org/sid/igmpproxy). If you do not trust the binary on this github, you can download it and extract it manually from the official debian package instead.

3. The default igmpproxy.conf uses eth9 (WAN2 SFP+ port) for the upstream network, and br70 (VLAN 70) for the downstream network. If you are using the Ethernet WAN port, or another VLAN for your TV, then modify igmpproxy.conf accordingly. 
    * **TIP:** To modify the config, run `vim igmpproxy.conf`. Press `i` to start editing in vim, navigate with your arrow keys and make your changes, press `ESC` to exit insert mode, type `:wq` to save and exit.
    * If you are using the Ethernet WAN port on the UDM Pro, change the two instances of eth9 to eth8.
    * If you are using the Ethernet WAN port on the UDM Base, change the two instances of eth9 to eth4. 
    * If you are using a different VLAN for TV traffic than VLAN 70, change the one instance of br70 to brX, where X is your VLAN number (br0 = default LAN).

4. Run igmpproxy in the foreground to test if everything is working.

    ```sh
    ./igmpproxy -nd ./igmpproxy.conf
    ```
  
5. On your wired Swisscom blueTV Box, tune to a channel and check if the TV is working without interruption for longer than 10 seconds. If igmpproxy is not working, your TV will stop working after 10 seconds.
	* If your box is stuck at the initialization screen, unplug it and replug it back in and wait a few minutes for it to start.
	
6. If the TV is working properly, press Ctrl+C in the SSH window to stop igmpproxy, then run it in the background via:

    ```sh
    ./igmpproxy ./igmpproxy.conf
    ```
  
7. igmpproxy will not start on boot by default. If you want to start igmpproxy at boot, [read the next section](#running-igmpproxy-on-boot).

### RUNNING IGMPPROXY ON BOOT

1. Set-up UDM Utilities Boot Script by following the instructions [here](https://github.com/boostchicken/udm-utilities/blob/master/on-boot-script/README.md). This boot script allows us to run igmpproxy on boot. 

2. Install the igmpproxy boot script.

    ```sh
    cd /mnt/data/on_boot.d
    ```
  
    ```sh
    curl -Lo 99-run-igmpproxy.sh https://raw.githubusercontent.com/bprskalo/udm-blueTV/main/run-igmpproxy.sh
    ```

    ```sh
    chmod +x 99-run-igmpproxy.sh
    ```
    
3. That's it, now igmpproxy will start automatically on boot. If you restart and want to check if it's running, you can run the command `ps aux | grep igmpproxy`. You should see a line that looks like this in the output if it's running: 

    ```sh
   6370 root     /mnt/data/igmpproxy/igmpproxy /mnt/data/igmpproxy/igmpproxy.conf
    ```


### CREDITS

This tutorial was adapted for the UDM from the USG TELUS TV tutorial by [peacey/udm-telus](https://github.com/peacey/udm-telus).

Many thanks to the extremely detailed tutorialas from [peacey](https://github.com/peacey/udm-telus), [fabian](https://github.com/fabianishere/udm-iptv) and [thilo](https://blog.thilojaeggi.ch/2021/12/bluetv-hinter-udm-pro-multicast.html) that made it a breeze to adapt and a lot of community entries from [Ubiquiti](https://community.ui.com) and [Swisscom](https://community.swisscom.ch).
