# Cisco-ISE-NAD-Template
Cisco Identity Service Engine - NAD Template for Wired Switches

!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!!!!! NAD TEMPLATE Wired by NS v 1.1  !!!!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!

!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!! Global Radius Commands  !!!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!

!!! Radius-KEY da cambiare a seconda delle password policy del cliente o aziendali
!!! 10.10.10.10 (ISE1)
!!! 10.10.10.11 (ISE2)


! enable authentication, authorization, and accounting on the access switch(es) by entering the following command
aaa new-model

! Create an authentication method for 802.1X
aaa authentication dot1x default group ISE

! Create an authorization method for 802.1X
aaa authorization network default group ISE

! Create an accounting method for 802.1X
aaa accounting dot1x default start-stop group ISE


!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!! START - Configure Radius Server for IOS Version 12.2  !!!!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!

! To conﬁgure the RADIUS server for a Cisco switch using IOS version 12.2
username radius-test password password

! Add Cisco ISE Servers to the radius group
radius-server host ISE1 auth-port 1812 acct-port 1813 test username "usernametobereplaced" key RADIUS-KEY
radius-server host ISE2 auth-port 1812 acct-port 1813 test username "usernametobereplaced" key RADIUS-KEY

!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!! END - Configure Radius Server for IOS Version 12.2  !!!!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!


!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!! START - Configure Radius Server for IOS Version 15 and above  !!!!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!

username radius-test secret 

! Add Cisco ISE Servers
radius server ISE1 address ipv4 10.10.10.10 auth-port 1812 acct-port 1813 key 0 RADIUS-KEY
radius server ISE2 address ipv4 10.10.10.11 auth-port 1812 acct-port 1813 key 0 RADIUS-KEY

!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!! END - Configure Radius Server for IOS Version 15 and above  !!!!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!


! Automate the Radius server availability  [username] need to be changed with username present on the ISE or on an ISE Identity Store
automate-tester username [username]

!! If you are using a Cisco 3850, 3650, 9300, 9500, or 5760 Series switch, the network access device sends its Calling-Station-Id in the
!! format of xxxx.xxxx.xxxx by default, instead of xx-xx-xx-xx-xx-xx. You can optionally change this with the radius-server attribute 31
!! mac format global conﬁguration command.

! Optional configure format for the switch to send the caling station id
radius-server attribute 31 mac format ietf lower-case

! Dead Criteria for wait 5 seconds and retry 3 times
radius-server dead-criteria time 5 tries 3


! Enable CoA Change of authorization
aaa server radius dynamic-author
! example client [ise_ip_address] server-key [shared_secret]
  client 10.10.10.10 server-key RADIUS-KEY
  client 10.10.10.11 server-key RADIUS-KEY


! Configure the switch to send Vendor Specific Attribute
!! In later versions of IOS 15.x and IOS XE, the VSAs are conﬁgured to send to the RADIUS server by default. In this case, these commandsmay 
!! not appear in the normal running conﬁguration after step 3.Certain default conﬁgurations are hidden in the normal showrunning-conﬁg output. 
!! In order to verify that VSAs are being sent tothe RADIUS server, issue the show running-conﬁg all | inc radiusservercommand.
radius-server vsa send authentication
radius-server vsa send accounting


! Enable Vendor Specific Attribute

! Identity the login request Service Type  Service-Type:Framed = 802.1X, • Service-Type:Call-Check = MAB, Service-Type:Login = Local WebAuth
radius-server attribute 6 on-for-login-auth


! Identify the IP of the Device if it's included or exists
radius-server attribute 8 include-in-access-req

! Server per inviare all'ISE class attribute cosi che sia disponibile nelle conditions dell'ISE
radius-server attribute 25 access-request include


! Force the switch to send traffic to the ISE always from the correct interface
ip radius source-interface [interface_name]
snmp-server trap-source [interface_name]
snmp-server source-interface informs [interface_name]



!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!! Global 802.1X Commands !!!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!

! Enable 802.1X Globally on the switch. command allows for the any source in the provided dACL to be replaced with the IP address of the single device connected to the switch port.
dot1x system-auth-control


! ip device tracking enable dACLs to function for IOS 12.2 and IOS 15.x
! In IOS Version 15.x, this command is enabled by default and may be seen in the output of show running-conﬁg all | inc ip device tracking.
ip device tracking

ip device tracking probe auto-source

ip device tracking probe delay 10

! check if the ip device tracking is working
show ip device tracking all


!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!! START - IP Device Tracking for IOS XE SISF Policy !!!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!

! Create a SISF Policy. Change [policy-name] with a custom name
device-tracking policy [policy-name]

! Enable tracking
tracking enable

! Enable device tracking to auto-source
device-tracking tracking auto-source


! Attach the created policy to the interface level. Change [policy-name] with the same name you assigned before
interface g1/0/1
  device-tracking attach-policy [policy-name]

! Check if it's working
show device-tracking database details

!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!! END - IP Device Tracking for IOS XE SISF Policy !!!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!



!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!! Common ACL for Deployments   !!!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!


!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
! For deployment in Monitor mode (This migh vary depending on the organization needs) for example include DHCP, TFTP for PXE Boot etc..
ip access-list extended ACL-ALLOW
 permit ip any any

!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
! For deployment in Low-Impact mode
ip access-list ext ACL-DEFAULT
 remark DHCP
 permit udp any eq bootpc any eq bootps 
 remark DNS
 permit udp any any eq domain
 remark Ping
 permit icmp any any
 remark PXE / TFTP
 permit udp any any eq tftp
 remark Drop all the rest 
 deny ip any any log


!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
! ACL for URL redirection with Web Auth
ip access-list ext ACLWEBAUTH-REDIRECT

remark explicitly deny DNS from being redirected to address a bug
deny udp any any eq 53

remark redirect all applicable traffic to the ISE Server
permit tcp any any eq 80
permit tcp any any eq 443

remark all other traffic will be implicitly denied from the redirection


!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
! ACL to be Used for URL redirection with a Posture agent (Example: Anyconnect)
ip access-list ext ACLAGENT-REDIRECT

remark explicitly deny DNS and DHCP from being redirected
deny udp any any eq 53 bootps


remark redirect HTTP traffic only
permit tcp any any eq 80

remark all other traffic will be implicitly denied from the redirection





!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!!   START - Interfaces Configuration Settings       !!!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!


! Interface Range command. Examples interface range fastethernet 5/1 - 5, gigabitethernet 1/1 - 2
interface range first_interface-last_interface
! check/ensure that you will be in a (config-if-range)


! Chek if the ports are in Layer 2
switchport

! configure ports as Layer 2 edge port
switchport host


!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!! Configure FlexAuth  !!!
!!!!!!!!!!!!!!!!!!!!!!!!!!!

! Configure authentication Priority
authentication priority dot1x mab

! Configure authentication method
authentication order dot1x mab

! Behaviour on authentication fail in an event of an access-reject message from ISE or Radius Server
! If a switch makes a local decision, such as assigning a VLAN instead
! of denying access, the switch no longer accepts CoA commands from
! the RADIUS server. Think about it this way: ISE now thinks that the
! switch port is down, but the switch has actually permitted the
! Endpoint to have access to the guest VLAN, so the state is now “out of sync.”

! Behaviour on authentication fail in an event of an access-reject message from ISE or Radius Server
authentication event fail action next-method


! Configure the port to use a local VLAN in case of Radius Server is dead. [vlan-id] need to be changed
authentication event server dead action authorize vlan [vlan-id]

! Configure the port to authorize Voice domain
authentication event server dead action authorize voice

! Reinitialize the port when the ISE is live again
authentication event server alive action reinitialize

! OPTIONAL 
authentication event server dead action reinitialize vlan [vlan-id]

! Configure HOST MODE on the port  (for Early Deploymentif you need to enforce is suggedest MDA)
authentication host-mode multi-auth

! Configure Violation action on the port
authentication violation restrict

! Configure Open Authentication
authentication open

! Enable MAB on the ports
mab

! Enable the ports to do IEE 802.1X Authentication even if you enabled globally the dot1x you must configure this settings on the ports 
dot1x pae authenticator

! Configure authentication timers. Leave all default except the tx-period
dot1x timeout tx-period 10


! Apply the pre-auth-authentication ACL
ip access-group ACL-ALLOW in

! turn on the authentication  (auto, force-authorized, force-unauthorized)
authentication port-control auto


!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!!   END - Interfaces Configuration Settings       !!!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!


!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!!!!!  Useful-Troubleshoot Commands   !!!!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!

! Check if the aaa servers are up
show aaa servers
!
test aaa show radius

! Test the aaa
test aaa group radius server 10.1.2.3 amolak password123 legacy

! Wrong password
test aaa group radius server 10.1.2.3 amolak wrongpassword legacy

!!! Cisco ASA  !!!

ASA-1# test aaa-server authentication RADIUS-SERVERS
Server IP Address or name: 10.1.2.3
Username: amolak
Password: password123
INFO: Attempting Authentication test to IP address <10.1.2.3> (timeout: 12 seconds)
INFO: Authentication Successful

ASA-1# test aaa-server authentication RADIUS-SERVERS
Server IP Address or name: 10.1.2.3
Username: amolak
Password: wrongpassword
INFO: Attempting Authentication test to IP address <10.1.2.3> (timeout: 12 seconds)
ERROR: Authentication Rejected: AAA failure


! Base command
show authentication session
! more detail on the specific interface
show authentication session interface [interface-name] detail


! Enable logging if required
CAT9300(config)#logging monitor informational
CAT9300(config)#logging origin-id ip
CAT9300(config)#logging source-interface <interface_id>
CAT9300(config)#logging host <ISE_MNT_PERSONA_IP_Address_x> transport udp port 20514


! Enable epm logging
epm logging


! WLC Debug 802.1x
debug dot1x

! debug client <mac-address>
debug client <mac-address>
