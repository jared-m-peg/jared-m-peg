# A Guide for Mikrotik RouterOS 7
Creating a new configuration, making it secure, and connecting to the internet.
Creating VLANs and isolating them from accessing each other (and isolating the Guest Wifi SSID from accessing the other VLANs).

## Part 1:
0. Create a new configuration
#Note: The purpose of creating a new config here was to bypass some issues that I had with creating VLANs on the default config that ships with the router. If you're having trouble with getting VLANs running on the default config, then this guide may help you get a fresh config running with VLANs working.

Log into Winbox, click on "New Terminal" on the menu on the left. Type out the following command and press enter.

```text
/system reset-configuration no-defaults=yes skip-backup=yes
```

This creates a new config making you disconnect from the current config. Ensure you backup your default config before creating a new one: Go to Files -> Backup, name your backup file, and click on Backup Config. This saves a backup file to your router which can be restored even after logging in to a new configuration.

Log back into Winbox after the fresh config is created. The default Login (username) is "admin" and the password is blank. Open a "New Terminal".

If you like, you can enable or disable Safe Mode by typing:
Ctrl + X
If using Safe Mode, ensure you exit Safe Mode to save any changes made to your config.

We will build this config by working in this order (correct dependency order): Layer 2 (bridge) -> Layer 3 (IP/DHCP) -> Security -> WAN -> Activation of Internet. For good baseline security, WAN will remain offline until the very end and firewall filter rules will be enabled before WAN is activated.

1. Create the LAN bridge

```text
/interface bridge
add name=bridge1
```

2. Add LAN ports to the bridge

```text
/interface bridge port
add bridge=bridge1 interface=ether3
add bridge=bridge1 interface=ether4
add bridge=bridge1 interface=ether5
```

Important note: I'm connected to Winbox via an ethernet cable on ether2. To prevent a lockout during this step, I don't add ether2 to bridge1; I add ether3, ether4, and ether5 to bridge1.

Verify:
```text
/interface bridge port print
```

3. Create interface lists
```text
/interface list
add name=WAN
add name=LAN
```

We will create additional interface lists later on which will run very well with multiple ethernet ports and VLANs.

4. Add interfaces to lists

WAN:

```text
/interface list member
add list=WAN interface=ether1
```

The Mikrotik router connects to my Huawei modem on ether1 using an ethernet cable. My Huawei modem is set to PPPoE passthrough/bridged mode (the Mikrotik does the routing instead of the Huawei).

LAN:

```text
/interface list member
add list=LAN interface=bridge1
```

5. Assign LAN IP address

```text
/ip address
add address=192.168.88.1/24 interface=bridge1
```

6. Reconnect to Winbox using the router's MAC address on ether3 instead of ether2 this time.

7. Create DHCP pool
```text
/ip pool
add name=LAN_POOL ranges=192.168.88.100-192.168.88.200
```

8. Create DHCP server
```text
/ip dhcp-server
add name=dhcp1 interface=bridge1 address-pool=LAN_POOL
```

9. Add DHCP network
```text
/ip dhcp-server network
add address=192.168.88.0/24 gateway=192.168.88.1 dns-server=1.1.1.1,8.8.8.8
```

10. Enable DHCP server
```text
/ip dhcp-server enable dhcp1
```

11. Create firewall rules
Input chain
```text
/ip firewall filter
add chain=input connection-state=established,related action=accept comment="Established Related"
add chain=input connection-state=invalid action=drop comment="Drop Invalid"
add chain=input protocol=icmp action=accept comment="Allow ICMP"
add chain=input in-interface-list=LAN action=accept comment="Allow LAN to manage router services"
```
Forward chain
```text
/ip firewall filter
add chain=forward connection-state=established,related action=accept comment="Established Related"
add chain=forward connection-state=invalid action=drop comment="Drop Invalid"
```
These rules provide a good level of protection before we connect to the internet. We will add final drop rules later on.

12. Create PPPoE client (disabled)
```text
/interface pppoe-client
add \
interface=ether1 \
name=pppoe-out1 \
user="username" \
password="password" \
add-default-route=yes \
use-peer-dns=yes \
disabled=yes
```

13. Add PPPoE to WAN list
```text
/interface list member
add list=WAN interface=pppoe-out1
```

14. Configure NAT
```text
/ip firewall nat
add chain=srcnat out-interface-list=WAN action=masquerade
```

This allows the router's DHCP server's addresses to be translated into the ISP IP address.

15. Complete firewall

Input drop:

```text
/ip firewall filter
add chain=input action=drop comment="Drop Everything Else"
```

Forward LAN -> WAN:

```text
/ip firewall filter
add chain=forward in-interface-list=LAN out-interface-list=WAN action=accept comment="LAN Internet Access"
```

Forward drop:

```text
/ip firewall filter
add chain=forward action=drop comment="Drop Everything Else"
```

16. Disable unnecessary services

```text
/ip service
disable telnet
disable ftp
disable www
disable api
disable api-ssl
```
Restrict WinBox:

```text
/ip service
set winbox address=192.168.88.0/24
```

Now Winbox can only be accessed from devices in my own subnet (192.168.88.0/24). I'm currently using a laptop to access Winbox on the address 192.168.88.200 that was assigned to my laptop by the DHCP Server that we created earlier.
- RESTRICTING WINBOX TO 192.168.88.0/24 IS PROBABLY NOT NECESSARY FOR SETTING THE ROUTER UP AT THIS STAGE BUT IT WAS THE STEP THAT I TOOK. WHEN THE VLANS ARE CREATED WINBOX CAN BE RESTRICTED TO VLAN10-MGMT VIA A FIREWALL FILTER RULE (INSTRUCTIONS WILL BE PROVIDED)

17. Verify everything
```text
/interface bridge print
/interface bridge port print
/ip address print
/ip dhcp-server print
/ip firewall filter print
/ip firewall nat print
```

```text
16. Enable PPPoE last
/interface pppoe-client
enable pppoe-out1
```

Verify:

```text
/interface pppoe-client monitor pppoe-out1 once
```

Test:

```text
/ping 1.1.1.1
```

## Part 2:

Design goals:

- VLAN10 = access to router services (Winbox, SSH) & access to internet
- VLAN20 = access to internet (cannot access router services) - used for wired CCTV devices, wired laptops, etc.
- VLAN30 = access to internet (cannot access router services) - used for wired laptops, wireless phones / laptops
- VLAN40 = access to internet (cannot access router services) - used for wireless phones (guests)
- All VLANs are isolated from each other
- Guest Wifi SSID runs on VLAN40 which is isolated from VLANs 10 / 20 / 30

VLAN 10	Management	ether2 (access port)
VLAN 20	Wired LAN	ether3 + ether4 (access ports)
VLAN 30	Main WiFi	WiFi SSID + ether5
VLAN 40	Guest WiFi	WiFi SSID only


1. Remove old LAN IP from bridge - DON'T USE STEP 1

First check:

```text
/ip address print
```

Remove:

```text
/ip address remove [find interface=bridge1]
```

DON'T DELETE THE IP ADDRESS / THE BRIDGE OTHERWISE YOU'LL LOSE INTERNET. NEED TO REMOVE THE IP ADDRESS LATER.

2. Create VLAN interfaces
```text
/interface vlan
add name=vlan10-mgmt interface=bridge1 vlan-id=10
add name=vlan20-lan interface=bridge1 vlan-id=20
add name=vlan30-wifi interface=bridge1 vlan-id=30
add name=vlan40-guest interface=bridge1 vlan-id=40
```

3. Create an IP Address for each VLAN

```text
/ip address
add address=192.168.10.1/24 interface=vlan10-mgmt
add address=192.168.20.1/24 interface=vlan20-lan
add address=192.168.30.1/24 interface=vlan30-wifi
add address=192.168.40.1/24 interface=vlan40-guest
```

4. Compare bridge1's IP address to the VLANs' IP addresses

```text
/ip address print
```

We can see that each vlan interface has its own gateway IP address (192.168.10.1/24, 192.168.20.1/24, etc.) as opposed to bridge1 that has 192.168.88.1./24. We currently have internet connectivity because of 192.168.88.1/24. We will later remove 192.168.88.1/24 from bridge1.

5. Configure Bridge Ports
We must configure the bridge ports before enabling VLAN filtering

```text
/interface bridge port
set [find interface=ether2] pvid=10
set [find interface=ether3] pvid=20
set [find interface=ether4] pvid=20
set [find interface=ether5] pvid=30
```

We'll add vlan40 (used for Guest Wifi) later on

6. Examine bridge ports & add the missing ether2

```text
/interface bridge port print
```

This command shows your bridge ports. The interface ether2 didn't appear for me; only ether3, ether4, and ether5 appeared. Therefore I needed to add the interface ether2 with this command:

```text
/interface bridge port
add bridge=bridge1 interface=ether2 pvid=10
```

7. Create DHCP server for VLAN 20

First create the DHCP pool:

```text
/ip pool
add name=pool-vlan20 ranges=192.168.20.100-192.168.20.200
```

Create the DHCP server:

```text
/ip dhcp-server
add name=dhcp-vlan20 interface=vlan20-lan address-pool=pool-vlan20 disabled=no
```

Add the DHCP network:

```text
/ip dhcp-server network
add address=192.168.20.0/24 gateway=192.168.20.1 dns-server=192.168.20.1
```

Then we also need to enable DNS forwarding (if it isn't already):

```text
/ip dns
set allow-remote-requests=yes
```

Ctrl + X to enter safe mode

```text
/interface bridge
set bridge1 vlan-filtering=yes
```

THIS STEP ENABLED VLAN FILTERING BUT INTERNET LOST ITS CONNECTION. NEEDED TO ADD FIREWALL FILTER RULES AND INTERFACE LISTS TO ALLOW VLANS TO ACCESS WAN/INTERNET.

1. Disable the old DHCP server (do not delete it)
```text
/ip dhcp-server disable dhcp1
```

2. Enable VLAN filtering
```text
/interface bridge set bridge1 vlan-filtering=yes
```
ADDED INTERFACE LISTS AND FIREWALL FILTER RULES TO ALLOW VLANS TO ACCESS WAN/INTERNET 
```text
/interface list
add name=VLAN20-LAN
add name=VLAN30-WIFI
add name=VLAN40-GUEST
```

```text
/interface list member
add list=VLAN20-LAN interface=vlan20-lan
add list=VLAN30-WIFI interface=vlan30-wifi
add list=VLAN40-GUEST interface=vlan40-guest
```

Forward rules to allow VLANs to access WAN:

```text
/ip firewall filter
add chain=forward action=accept in-interface-list=VLAN20-LAN out-interface-list=WAN comment="VLAN20 Internet Access"
add chain=forward action=accept in-interface-list=VLAN30-WIFI out-interface-list=WAN comment="VLAN30 Internet Access"
add chain=forward action=accept in-interface-list=VLAN40-GUEST out-interface-list=WAN comment="VLAN40 Internet Access"
```

Go to IP -> Firewall -> Filter Rules and drag these three forward chain rules above the final drop rule on the forward chain, but position them lower than the "drop invalid" rule.

DO THE INPUT RULES LATER - MGMT WILL BE THE ONLY ONE ALLOWED ACCESS TO ROUTER SERVICES.

Now try disabling dhcp1, enabling vlan filtering, and reconnecting to the router on 192.168.20.x or the router's MAC address:

```text
/ip dhcp-server disable dhcp1
/interface bridge set bridge1 vlan-filtering=yes
```

After reconnecting, use these two commands:

```text
/interface bridge print
```

should show:

vlan-filtering=yes

and:

```text
/ip dhcp-server lease print
```

should show your PC on dhcp-vlan20 with a 192.168.20.x address.

Remove the old bridge DHCP server:

```text
/ip dhcp-server remove dhcp1
```

Remove the old bridge IP address:

```text
/ip address remove [find address="192.168.88.1/24"]
```

Remove 192.168.88.0/24 here as well:
First check your addresses:

```text
/ip dhcp-server network print
```

This prints something like the following:

ADDRESS			GATEWAY
0 192.168.20.0/24    192.168.20.1
1 192.168.88.0/24    192.168.88.1

Remove 192.168.88.0/24 by its number:

```text
/ip dhcp-server network remove 1
```

Under IP -> DHCP Server -> Networks, double-click on 192.168.20.0/24 and add the following DNS servers:
9.9.9.9
149.112.112.112

The firewall filter forward rule for LAN can now be removed:
```text
/ip firewall filter print
/ip firewall filter remove 10
```

Check your VLANs:

```text
/interface vlan print
```

You should see:

vlan10   vlan-id=10   interface=bridge1
vlan20   vlan-id=20   interface=bridge1
vlan30   vlan-id=30   interface=bridge1
vlan40   vlan-id=40   interface=bridge1

Check your addresses: 

```text
/ip address print
```

You should see:

192.168.10.1/24 vlan10-mgmt
192.168.20.1/24 vlan20-lan
192.168.30.1/24 vlan30-wifi
192.168.40.1/24 vlan40-guest

Create VLAN10-MGMT--this VLAN will be used to manage router services (Winbox / SSH):

```text
/interface list add name=VLAN10-MGMT
```

Then add VLAN10 to it:

```text
/interface list member add list=VLAN10-MGMT interface=vlan10-mgmt
```

Check:

```text
/interface list print
```

and:

```text
/interface list member print
```

Add the following firewall filter rule:

```text
/ip firewall filter add chain=input action=accept in-interface-list=VLAN10-MGMT comment="Allow MGMT to manage router services"
```

Note: Go to IP -> Firewall -> Filter Rules, and drag this rule above the "Drop everything else" input rule. This can also be done via the CLI but using the CLI here is more cumbersome.

Create pools

```text
/ip pool
add name=pool-vlan10 ranges=192.168.10.100-192.168.10.200
add name=pool-vlan30 ranges=192.168.30.100-192.168.30.200
add name=pool-vlan40 ranges=192.168.40.100-192.168.40.200
```

Verify:

```text
/ip pool print
```

Create DHCP servers

```text
/ip dhcp-server
add name=dhcp-vlan10 interface=vlan10-mgmt address-pool=pool-vlan10 disabled=no
add name=dhcp-vlan30 interface=vlan30-wifi address-pool=pool-vlan30 disabled=no
add name=dhcp-vlan40 interface=vlan40-guest address-pool=pool-vlan40 disabled=no
```

Verify:

```text
/ip dhcp-server print
```

You should have:

```text
dhcp-vlan10
dhcp-vlan20
dhcp-vlan30
dhcp-vlan40
```

Add DHCP network entries

```text
/ip dhcp-server network
add address=192.168.10.0/24 gateway=192.168.10.1 dns-server=9.9.9.9,149.112.112.112
add address=192.168.30.0/24 gateway=192.168.30.1 dns-server=9.9.9.9,149.112.112.112
add address=192.168.40.0/24 gateway=192.168.40.1 dns-server=9.9.9.9,149.112.112.112
```

Then check:

```text
/ip dhcp-server network print
```

The router is now ready to hand out addresses to all VLANs

Under Bridge -> Ports double click on each interface (ether2/3/4/5), go to the VLAN tab, and set Frame Types to:
admit only untagged and priority tagged

After this we move to the bridge VLAN table cleanup, because your current VLAN table is still missing the access assignments for:

ether2 → VLAN10
ether4 → VLAN20
ether5 → VLAN30

and then we configure the WiFi SSIDs.

Fix Bridge VLAN table - NOT NECESSARY

/interface bridge vlan
add bridge=bridge1 vlan-ids=10 tagged=bridge1 untagged=ether2
add bridge=bridge1 vlan-ids=20 tagged=bridge1 untagged=ether3,ether4
add bridge=bridge1 vlan-ids=30 tagged=bridge1 untagged=ether5
add bridge=bridge1 vlan-ids=40 tagged=bridge1

The above may not work, so run these commands to add ether ports to the VLANs - MIGHT ALSO BE UNECESSARY

/interface bridge vlan
set [find vlan-ids=10] tagged=bridge1 untagged=ether2
set [find vlan-ids=20] tagged=bridge1 untagged=ether3,ether4
set [find vlan-ids=30] tagged=bridge1 untagged=ether5

Allow VLAN10-MGMT to access the internet

```text
/ip firewall filter add chain=forward action=accept in-interface-list=VLAN10-MGMT out-interface-list=WAN comment="VLAN10 Internet Access"
```

Disable inter-VLAN traffic through firewall filter rules
We will prevent all VLANs from accessing the other VLANs to increase security

```text
/ip firewall filter add chain=forward action=drop in-interface-list=VLAN30-WIFI out-interface-list=VLAN10-MGMT comment="Block WIFI access to MGMT"
/ip firewall filter add chain=forward action=drop in-interface-list=VLAN30-WIFI out-interface-list=VLAN20-LAN comment="Block WIFI access to LAN"
/ip firewall filter add chain=forward action=drop in-interface-list=VLAN30-WIFI out-interface-list=VLAN40-GUEST comment="Block WIFI access to GUEST"

/ip firewall filter add chain=forward action=drop in-interface-list=VLAN20-LAN out-interface-list=VLAN10-MGMT comment="Block LAN accessto MGMT"
/ip firewall filter add chain=forward action=drop in-interface-list=VLAN20-LAN out-interface-list=VLAN30-WIFI comment="Block LAN access to WIFI"
/ip firewall filter add chain=forward action=drop in-interface-list=VLAN20-LAN out-interface-list=VLAN40-GUEST comment="Block LAN access to GUEST"

/ip firewall filter add chain=forward action=drop in-interface-list=VLAN10-MGMT out-interface-list=VLAN20-LAN comment="Block MGMT access to LAN"
/ip firewall filter add chain=forward action=drop in-interface-list=VLAN10-MGMT out-interface-list=VLAN30-WIFI comment="Block MGMT access to WIFI"
/ip firewall filter add chain=forward action=drop in-interface-list=VLAN10-MGMT out-interface-list=VLAN40-GUEST comment="Block MGMT access to GUEST"

/ip firewall filter add chain=forward action=drop in-interface-list=VLAN40-GUEST out-interface-list=VLAN10-MGMT comment="Block GUEST access to MGMT"
/ip firewall filter add chain=forward action=drop in-interface-list=VLAN40-GUEST out-interface-list=VLAN20-LAN comment="Block GUEST access to LAN"
/ip firewall filter add chain=forward action=drop in-interface-list=VLAN40-GUEST out-interface-list=VLAN30-WIFI comment="Block GUEST access to WIFI"
```

Under IP -> Firewall -> Filter Rules, drag the forward chain rule "Drop Everything Else" (final drop rule) all the way to the bottom.

Allow only VLAN10-MGMT to manage router services (Winbox / SSH)
We want only devices on ether2 (VLAN10-MGMT) to access router services (Winbox / SSH), not ether3 (VLAN20-LAN), so disconnect your PC from ether3 (VLAN20-LAN) and reconnect to Winbox on ether2 (VLAN10-MGMT)
After connecting to ether2 (VLAN10-MGMT), find the number for the "Allow LAN to manage router services" rule (the input rule for VLAN20-LAN)

```text
/ip firewall filter print
```

and remove it

```text
/ip firewall filter remove 4
```

Now only devices on VLAN10-MGMT can access router services (Winbox / SSH).

Our firewall filter rules should now look like this:

```text
/ip/firewall/filter> print

 0    ;;; Allow Established Related
      chain=input action=accept connection-state=established,related log=no log-prefix="" 

 1    ;;; Drop Invalid
      chain=input action=drop connection-state=invalid 

 2    ;;; Allow ICMP
      chain=input action=accept protocol=icmp 

 3    ;;; Allow MGMT to manage router services
      chain=input action=accept in-interface-list=VLAN10-MGMT 

 4    ;;; Drop Everything Else
      chain=input action=drop 

 5    ;;; Allow Established Related
      chain=forward action=accept connection-state=established,related log=no log-prefix="" 

 6    ;;; Drop Invalid
      chain=forward action=drop connection-state=invalid 

 7    ;;; VLAN30 Internet Access
      chain=forward action=accept in-interface-list=VLAN30-WIFI out-interface-list=WAN 

 8    ;;; VLAN20 Internet Access
      chain=forward action=accept in-interface-list=VLAN20-LAN out-interface-list=WAN 

 9    ;;; VLAN10 Internet Access
      chain=forward action=accept in-interface-list=VLAN10-MGMT out-interface-list=WAN 

10    ;;; VLAN40 Internet Access
      chain=forward action=accept in-interface-list=VLAN40-GUEST out-interface-list=WAN 

11    ;;; Block WIFI access to MGMT
      chain=forward action=drop in-interface-list=VLAN30-WIFI out-interface-list=VLAN10-MGMT 

12    ;;; Block WIFI access to LAN
      chain=forward action=drop in-interface-list=VLAN30-WIFI out-interface-list=VLAN20-LAN 

13    ;;; Block WIFI access to GUEST
      chain=forward action=drop in-interface-list=VLAN30-WIFI out-interface-list=VLAN40-GUEST 

14    ;;; Block LAN accessto MGMT
      chain=forward action=drop in-interface-list=VLAN20-LAN out-interface-list=VLAN10-MGMT 

15    ;;; Block LAN access to WIFI
      chain=forward action=drop in-interface-list=VLAN20-LAN out-interface-list=VLAN30-WIFI 

16    ;;; Block LAN access to GUEST
      chain=forward action=drop in-interface-list=VLAN20-LAN out-interface-list=VLAN40-GUEST 

17    ;;; Block MGMT access to LAN
      chain=forward action=drop in-interface-list=VLAN10-MGMT out-interface-list=VLAN20-LAN 

18    ;;; Block MGMT access to WIFI
      chain=forward action=drop in-interface-list=VLAN10-MGMT out-interface-list=VLAN30-WIFI 

19    ;;; Block MGMT access to GUEST
      chain=forward action=drop in-interface-list=VLAN10-MGMT out-interface-list=VLAN40-GUEST 

20    ;;; Block GUEST access to MGMT
      chain=forward action=drop in-interface-list=VLAN40-GUEST out-interface-list=VLAN10-MGMT 

21    ;;; Block GUEST access to LAN
      chain=forward action=drop in-interface-list=VLAN40-GUEST out-interface-list=VLAN20-LAN 

22    ;;; Block GUEST access to WIFI
      chain=forward action=drop in-interface-list=VLAN40-GUEST out-interface-list=VLAN30-WIFI 

23    ;;; Drop Everything Else
      chain=forward action=drop
```

To look at it more simply:

```text
Input chain
0  established/related		ALLOW
1  invalid			DROP
2  ICMP				ALLOW
3  VLAN10-MGMT -> router	ALLOW
4  everything else		DROP
```

VLAN10-MGMT can initiate connections with the router because it has been explicity allowed to do so
Any attempted connections to the router not explicitly allowed are blocked
Thus, VLAN20/30/40 cannot make connections to the router since they have not been explicitly allowed to do so

```text
Forward chain
5  established/related		ALLOW
6  invalid			DROP

7–10  VLANs -> WAN		ALLOW

11–22  VLAN <--> VLAN		DROP

23  everything else		DROP
```

All VLANs can access the internet
VLAN to VLAN connections are blocked: i.e., VLAN20-LAN cannot initiate a connection to VLAN30-WIFI and vice-versa


## Setting up WiFi 5Ghz

In Winbox, click on WiFi -> WiFi and then double-click on wifi2
In the Configuration tab name your SSID and Country
In the Security tab click on Authentication types and tick WPA2 PSK
In the Security tab click Passphrase and type a password (used for connecting to the router's WiFi)
In the Datapath tab tick/check the Client Isolation box (devices connected to the WiFi network will be isolated from each other)

Go to Bridge -> Ports, click on New, and under General tab set Interface to wifi2
Under VLAN tab set PVID to 30

Now go to Bridge -> VLANs, double-click on 30 (in the VLAN IDs column), add wifi2 to Untagged (both wifi2 and ether5 will be untagged)

Create a wifi guest network for that will be isolated on VLAN40 from the other VLANs
Go to WiFi, click on New
Set Master to wifi2
Set Name to wifi2-guest (or something similar)
Set Mode to ap
Under Configuration tab name the SSID of your guest network
Under Datapath tab tick/check the Client Isolation box
Now go to Bridge -> VLANs, double-click on 40 (in the VLAN IDs column), add wifi2-guest to Untagged
Go to Bridge -> Ports, under General tab set Interface to wifi2-guest, and under VLAN tab set PVID to 40
Go to WiFi, double-click on wifi2-guest and tick the Enabled box
After connecting to the guest network with a laptop/phone you can see the DHCP server lease with:
```text
/ip dhcp-server lease print
```
it should print something like 192.168.40.200

Under Bridge -> Ports double click on each interface (ether2/3/4/5, wifi2, wifi2-guest), go to the VLAN tab, and set Frame Types to:
admit only untagged and priority tagged

## CHECKING VLAN ACCESS TO ROUTER SERVICES
We need to verify that VLAN10-MGMT is the only vlan that can access router services. To do so, first connect your PC to the router on ether2 and attempt a Winbox login.
Then attempt to login to Winbox on ether3, ether4, ether5, wifi2, and wifi2-guest. It shouldn't work.

## CHECKING INTER-VLAN ACCESS
Devices connected to one VLAN shouldn't be able to access devices connected to another VLAN.
Connect your PC to ether2 (VLAN10-MGMT) and try communicating with a device (PC, phone) that is connected to VLAN30-WIFI by running tests in CMD and Powershell. 
Under IP -> DHCP Server -> Leases identify the DHCP lease (e.g. 192.168.30.200) of a device (PC, phone) on VLAN30-WIFI other than your PC that is connected to VLAN10-MGMT. Try using your PC (on VLAN10-MGMT) to communicate with another device (on VLAN30-WIFI) by using the following tests on Powershell on Windows 10:
```text
Test-NetConnection 192.168.30.243 -Port 80
Test-NetConnection 192.168.30.243 -Port 445
```
In Winbox CLI, we can use
```text
/ip firewall filter print stats
```
to monitor an increase of blocked packets

;;; Block MGMT access to WIFI
26   forward  drop                          1576       24

After running these Powershell tests, the packets increased from 0 to 24, which means traffic from VLAN10-MGMT to VLAN30-WIFI is being blocked (24 packets were blocked)

Under Bridge -> Ports double click on each interface (ether2/3/4/5, wifi2, wifi2-guest), go to the VLAN tab, and set Frame Types to:
admit only untagged and priority tagged



