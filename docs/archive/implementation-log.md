# Implementation History

[Back to main README](../../readme.md) · [Current address plan](../addressing.md) · [Rebuild guide](../guide.md)

> English edition of the October 26–27, 2026 build record. Old settings are kept as history, not as instructions for a new build. All 41 command/output blocks from the earlier archive are kept below.
>
> The original mixed-language snapshot remains in [Git history](https://github.com/junuiui/network-infrastructure-lab/blob/9cf695e292b23069f3d2f3c1cd7df35ceccec8ea/docs/archive/implementation-log.md). Use the current topic pages for the final state.

## 1. Original company and address plan

The original company plan used 70 employees: HR/Finance 8, Engineering 35, Sales/Marketing 22, and IT 5. The initial device target listed 172 items before the approximate Guest group. These were planning counts, not a claim that all devices were placed.

| Department | Original target |
| :--- | :--- |
| HR/Finance | 8 PCs, 8 phones, 2 printers |
| Engineering | 35 PCs/laptops, 35 phones, 2 test workstations |
| Sales/Marketing | 22 PCs/laptops, 22 phones, 1 printer |
| IT | 5 management PCs, 5 phones, 1 admin terminal |
| Server Farm | AD/DNS, DHCP, Web, FTP, Database, Syslog |
| Guest | About 20 temporary wireless devices |

The AD, FTP, and Database items were targets, not completed services. Server-PT is not Windows Server/AD.

Original VLSM table:
| Subnet Name | VLAN ID | Required Hosts | Subnet Mask | Network Address | Usable IP Range | Gateway IP |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **ENG-Data** | 20 | 37 | /26 (255.255.255.192) | 192.168.0.0 | 192.168.0.1 ~ 192.168.0.62 | 192.168.0.1 |
| **ENG-Voice** | 120 | 35 | /26 (255.255.255.192) | 192.168.0.64 | 192.168.0.65 ~ 192.168.0.126 | 192.168.0.65 |
| **S/M-Data** | 30 | 23 | /27 (255.255.255.224) | 192.168.0.128 | 192.168.0.129 ~ 192.168.0.158 | 192.168.0.129 |
| **S/M-Voice** | 130 | 22 | /27 (255.255.255.224) | 192.168.0.160 | 192.168.0.161 ~ 192.168.0.190 | 192.168.0.161 |
| **Guests** | 99 | 20 | /27 (255.255.255.224) | 192.168.0.192 | 192.168.0.193 ~ 192.168.0.222 | 192.168.0.193 |
| **HR-Data** | 10 | 10 | /28 (255.255.255.240) | 192.168.0.224 | 192.168.0.225 ~ 192.168.0.238 | 192.168.0.225 |
| **HR-Voice** | 110 | 8 | /28 (255.255.255.240) | 192.168.0.240 | 192.168.0.241 ~ 192.168.0.254 | 192.168.0.241 |
| **Server Farm** | 50 | 6 | /29 (255.255.255.248) | 192.168.1.0 | 192.168.1.1 ~ 192.168.1.6 | 192.168.1.1 |
| **IT-Data** | 40 | 6 | /29 (255.255.255.248) | 192.168.1.8 | 192.168.1.9 ~ 192.168.1.14 | 192.168.1.9 |
| **IT-Voice** | 140 | 5 | /29 (255.255.255.248) | 192.168.1.16 | 192.168.1.17 ~ 192.168.1.22 | 192.168.1.17 |


Data/Voice VLANs were HR 10/110, ENG 20/120, S/M 30/130, and IT 40/140. Server used 50 and Guest 99. Staff VLAN 60 came later.

## 2. Initial layout

The early layout put the 2911 above the 3560 core, with access switches and endpoint groups below. The core handled inter-VLAN routing and relay. External services were still planned.

![Early department-to-core links](../../resources/2-main-switch-connect.png)

*Build in progress. Some links show transitional states.*

![Early edge-router connection](../../resources/3-router.png)

*The edge was connected before its IP/routing configuration was added.*

## 3. Initial switch commands

These blocks use the **old** addresses.

### Access example: HR
```text
enable
configure terminal

! VLAN Creation
vlan 10
 name HR-Data
exit

vlan 110
 name HR-Voice
exit

! Access Port Configuration (PC & IP Phone)
interface range fa0/1 - 2
 switchport mode access
 switchport access vlan 10
 switchport voice vlan 110
exit

! Printer Port Configuration
interface fa0/3
 switchport mode access
 switchport access vlan 10
exit

end
write memory
```

### Core VLANs
```text
enable
configure terminal

vlan 10
 name HR-Data
vlan 110
 name HR-Voice
vlan 20
 name ENG-Data
vlan 120
 name ENG-Voice
vlan 30
 name S/M-Data
vlan 130
 name S/M-Voice
vlan 40
 name IT-Data
vlan 140
 name IT-Voice
vlan 50
 name Server-Farm
vlan 99
 name Guests
exit
```

### Original core SVI gateways
```text
configure terminal
ip routing

! ENG-Data & Voice
interface vlan 20
 ip address 192.168.0.1 255.255.255.192
 no shutdown
exit
interface vlan 120
 ip address 192.168.0.65 255.255.255.192
 no shutdown
exit

! S/M-Data & Voice
interface vlan 30
 ip address 192.168.0.129 255.255.255.224
 no shutdown
exit
interface vlan 130
 ip address 192.168.0.161 255.255.255.224
 no shutdown
exit

! Guests
interface vlan 99
 ip address 192.168.0.193 255.255.255.224
 no shutdown
exit

! HR-Data & Voice
interface vlan 10
 ip address 192.168.0.225 255.255.255.240
 no shutdown
exit
interface vlan 110
 ip address 192.168.0.241 255.255.255.240
 no shutdown
exit

! IT-Data & Voice
interface vlan 40
 ip address 192.168.1.9 255.255.255.248
 no shutdown
exit
interface vlan 140
 ip address 192.168.1.17 255.255.255.248
 no shutdown
exit

! Server Farm Gateway
interface vlan 50
 ip address 192.168.1.1 255.255.255.248
 no shutdown
exit
```

### Original direct DHCP-server port
```text
configure terminal
interface gigabitEthernet 0/2
 switchport mode access
 switchport access vlan 50
 no shutdown
exit
```

Core Gi0/2 was used here. It was later left unused when the DHCP server moved to Server-Switch Fa0/1.

### Original relay settings
```text
configure terminal

interface vlan 10
 ip helper-address 192.168.1.2
interface vlan 110
 ip helper-address 192.168.1.2
interface vlan 20
 ip helper-address 192.168.1.2
interface vlan 120
 ip helper-address 192.168.1.2
interface vlan 30
 ip helper-address 192.168.1.2
interface vlan 130
 ip helper-address 192.168.1.2
interface vlan 40
 ip helper-address 192.168.1.2
interface vlan 140
 ip helper-address 192.168.1.2
interface vlan 50
 ip helper-address 192.168.1.2
interface vlan 99
 ip helper-address 192.168.1.2

end
write memory
```

The initial config included a helper on VLAN 50. It was not needed because the DHCP server was in that VLAN.

## 4. Early DHCP troubleshooting

Clients showed APIPA or waited for a lease.

1. Layer 3 commands had been entered on an access switch. The work moved to the core 3560.
2. VLAN 99 and VLAN 10 used network addresses .0.192 and .0.224 as gateways. They were corrected to usable .193 and .225 in the original plan.
3. The direct DHCP-server port was in HR VLAN 10 rather than Server VLAN 50. Its access VLAN was corrected.
4. VLAN 50, 40, and 140 had no SVI addresses. The missing gateways prevented normal relay/reply paths.

The sample fix for VLAN 50 was:
```text
  interface vlan 50
   ip address 192.168.1.1 255.255.255.248
   no shutdown
```

The early log reported successful unicast relay to 192.168.1.2 and correct client leases after those changes. Later tests are recorded separately.

## 5. October 26: IT, Server Farm, Guest, and Wi-Fi

### Work and evidence at that time

IT, Server Farm, and Guest wired setup were reported complete. Guest wireless setup was also reported complete, but individual test outputs and the latest .pkt had not yet been collected.

The listed commands were reproduction steps, not a complete record of every command typed.

| Work | Reason | Method |
| :--- | :--- | :--- |
| Core trunks | Carry department VLANs | Explicit trunk setup was proposed |
| IT section | Separate admin clients | VLAN 40, two PCs, one printer |
| IT Voice | Prepare phone network | VLAN 140 and trunk allowance |
| Server Farm | Separate server connections | Server switch and VLAN 50 |
| Central DHCP | Keep one address service | Core helper to .1.2 |
| Guest wired | Separate visitors | VLAN 99 and Guest switch |
| Guest ACL | Block internal access | DHCP/gateway exceptions before private-network denies |
| DHCP cleanup | Avoid static/broadcast conflicts | Limit default pool and HR user counts |
| Guest Wi-Fi | Apply the Guest policy to wireless | AP on Access VLAN 99 |

The port map was Core Fa0/1–3 to HR/ENG/S/M, Fa0/4 to IT, Fa0/5 to Server, and Fa0/6 to Guest. Each access uplink used Gi0/1. Core Gi0/1 was physically linked to Router Gi0/0, with external routing still unfinished.

### Core setup recorded that day
```text
enable
configure terminal
ip routing

vlan 40
 name IT-Data
exit
vlan 140
 name IT-Voice
exit
vlan 50
 name Server-Farm
exit
vlan 99
 name Guest
exit

interface range fastEthernet0/1 - 3
 switchport trunk encapsulation dot1q
 switchport mode trunk
 no shutdown
exit

interface fastEthernet0/4
 description TO-IT-SWITCH
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 40,140
 no shutdown
exit

interface fastEthernet0/5
 description TO-SERVER-SWITCH
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 50
 no shutdown
exit

interface fastEthernet0/6
 description TO-GUEST-SWITCH
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 99
 no shutdown
exit

interface vlan 40
 ip address 192.168.1.9 255.255.255.248
 ip helper-address 192.168.1.2
 no shutdown
exit

interface vlan 50
 ip address 192.168.1.1 255.255.255.248
 no shutdown
exit

interface vlan 99
 ip address 192.168.0.193 255.255.255.224
 ip helper-address 192.168.1.2
 no shutdown
exit

end
copy running-config startup-config
```

### IT access switch
```text
enable
configure terminal
hostname IT-Switch

vlan 40
 name IT-Data
exit
vlan 140
 name IT-Voice
exit

interface range fastEthernet0/1 - 3
 switchport mode access
 switchport access vlan 40
 spanning-tree portfast
 no shutdown
exit

interface gigabitEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan 40,140
 no shutdown
exit

end
copy running-config startup-config
```

The initial IT PCs connected directly. The sample config did not yet put Voice VLAN 140 on those access ports. Phone ports needed it later. IT printer was .1.14/29 with gateway .1.9.

### Server access switch
```text
enable
configure terminal
hostname Server-Switch

vlan 50
 name Server-Farm
exit

interface range fastEthernet0/1 - 2
 switchport mode access
 switchport access vlan 50
 spanning-tree portfast
 no shutdown
exit

interface gigabitEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan 50
 no shutdown
exit

end
copy running-config startup-config
```

DHCP .1.2 and Web .1.3 used /29 and gateway .1.1. DHCP and HTTP services were turned on.

### Guest wired switch
```text
enable
configure terminal
hostname Guest-Switch

vlan 99
 name Guest
exit

interface range fastEthernet0/1 - 2
 switchport mode access
 switchport access vlan 99
 spanning-tree portfast
 no shutdown
exit

interface gigabitEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan 99
 no shutdown
exit

end
copy running-config startup-config
```

### Pool corrections at that stage

| Pool | Start | Mask | Gateway | Max users |
| :--- | :--- | :--- | :--- | ---: |
| IT-Data | 192.168.1.10 | 255.255.255.248 | 192.168.1.9 | 4 |
| IT-Voice | 192.168.1.18 | 255.255.255.248 | 192.168.1.17 | 4 |
| Guests | 192.168.0.194 | 255.255.255.224 | 192.168.0.193 | 28 |
| HR-Data | 192.168.0.226 | 255.255.255.240 | 192.168.0.225 | 13 |
| HR-Voice | 192.168.0.242 | 255.255.255.240 | 192.168.0.241 | 13 |

HR .239 and .255 were broadcast addresses, not leases. Server-Farm DHCP was removed because servers used static IPs. If default serverPool could not be removed, the workaround was .1.6/29, gateway .1.1, one lease. That address had to stay out of static assignments.

### Guest ACL: original addresses
```text
enable
configure terminal

ip access-list extended GUEST-IN
 permit udp any eq 68 any eq 67
 permit icmp 192.168.0.192 0.0.0.31 host 192.168.0.193 echo
 deny ip any 10.0.0.0 0.255.255.255
 deny ip any 172.16.0.0 0.15.255.255
 deny ip any 192.168.0.0 0.0.255.255
 permit ip 192.168.0.192 0.0.0.31 any
exit

interface vlan 99
 ip access-group GUEST-IN in
exit

end
copy running-config startup-config
```

It allowed DHCP and gateway Ping, then blocked private destinations. The final permit did not create Internet connectivity. This was not same-VLAN client isolation or a stateful firewall. Internal DNS had no exception.

### Guest Wi-Fi setup

AccessPoint-PT, not LAP-PT/WLC, was connected to Guest Fa0/3 using Port 0. Laptop-PT used WPC300N.

```text
enable
configure terminal

interface fastEthernet0/3
 description TO-GUEST-AP
 switchport mode access
 switchport access vlan 99
 spanning-tree portfast
 no shutdown
exit

end
copy running-config startup-config
```

AP settings were SSID FakeCompany-Guest, channel 6, WPA2-PSK, AES, and a lab-only 8–63-character passphrase. Both wired and wireless AP ports needed to be On. The key was not written into the document.

Laptop setup: power Off, replace the wired module with WPC300N, power On, connect in PC Wireless, then choose DHCP on Wireless0. Config → Wireless0 was an alternative if the PC Wireless menu differed.

Expected address at that time: .0.194–221 /27, gateway .0.193.

Client checks:
```text
ipconfig /all
ping 192.168.0.193
ping 192.168.1.2
ping 192.168.1.3
```

Core checks:
```text
show access-lists
show ip interface brief
show interfaces trunk
```

Guest switch checks:
```text
show vlan brief
show interfaces fastEthernet0/3 switchport
show mac address-table
```

The expected tests were SSID connection, DHCP, gateway Ping success, server Ping/HTTP blocks, DHCP renewal, and rejection of a wrong PSK. Individual results were still unknown at that stage. Later new-address Ping results did not prove the wrong-PSK test was run.

| Symptom | Check |
| :--- | :--- |
| SSID missing | AP wireless On, range, laptop power/module |
| Connection fails | SSID, WPA2/AES, matching key |
| Wireless0 missing | Power Off and check WPC300N |
| APIPA after association | Access VLAN, trunk, helper, pool, DHCP ACL rule |
| Internal access succeeds | Guest ACL attachment and order |
| External access fails | External routing/PAT was not built yet |

Save the switch startup config and .pkt. Recheck DHCP and ACL before continuing.

## 6. Earlier remaining-work list

At the end of October 26, remaining work was wireless evidence, address capacity, SSH, external routing/PAT, DNS, phone registration/calls, selected server services, backup, monitoring, and final documents.

The capacity review found:

- Server /29: six usable IPs, only five after the gateway, short for six servers.
- IT Data /29: short for five PCs plus an admin terminal.
- IT Voice /29: enough for five phones plus gateway, but the pool allowed only four.
- ENG printer .0.4 overlapped DHCP .2–61.

The log also set boundaries: Server-PT was not AD, simulated external access was not public Internet, and external Netmiko/PRTG/Zabbix integration was not assumed. The old README's “80% reduction” claim had no measurement and was later removed.

References: [AP configuration](https://tutorials.ptnetacad.net/help/default/config_others.htm), [Laptop modules](https://tutorials.ptnetacad.net/help/default/devicesAndModules_endDevices.htm).

## 7. October 27: Address expansion

### Decision and final plan

The new rule was 50% more devices plus the gateway. HR Voice already met this with /28 but was expanded to /27 by choice.

The final old/new table and pools are in [Addressing](../addressing.md). That page keeps the old table above as well.

The move order was IT, Server, S/M, Guest, HR Data, then HR Voice. This freed each overlapping range before the next change.

### Migration commands
```text
enable
configure terminal

! 1. Move IT first
interface vlan 40
 no ip address
exit
interface vlan 140
 no ip address
exit
interface vlan 40
 ip address 192.168.1.17 255.255.255.240
 no shutdown
exit
interface vlan 140
 ip address 192.168.1.33 255.255.255.240
 no shutdown
exit

! 2. Expand Server Farm
interface vlan 50
 ip address 192.168.1.1 255.255.255.240
exit

! 3. Move Sales/Marketing
interface vlan 30
 no ip address
exit
interface vlan 130
 no ip address
exit
interface vlan 30
 ip address 192.168.1.65 255.255.255.192
 no shutdown
exit
interface vlan 130
 ip address 192.168.1.129 255.255.255.192
 no shutdown
exit

! 4. Move Guest and replace ACL while SVI is down
interface vlan 99
 shutdown
 no ip address
 ip address 192.168.1.193 255.255.255.192
exit
no ip access-list extended GUEST-IN
ip access-list extended GUEST-IN
 permit udp any eq 68 any eq 67
 permit icmp 192.168.1.192 0.0.0.63 host 192.168.1.193 echo
 deny ip any 10.0.0.0 0.255.255.255
 deny ip any 172.16.0.0 0.15.255.255
 deny ip any 192.168.0.0 0.0.255.255
 permit ip 192.168.1.192 0.0.0.63 any
exit
interface vlan 99
 ip access-group GUEST-IN in
 no shutdown
exit

! 5. Move HR Data before expanding HR Voice
interface vlan 10
 no ip address
 ip address 192.168.0.193 255.255.255.224
 no shutdown
exit
interface vlan 110
 no ip address
 ip address 192.168.0.225 255.255.255.224
 no shutdown
exit

end
show ip interface brief
show access-lists
```

These are migration steps, not a blank-file build. DHCP, static clients, server NIC masks, and Guest ACL needed matching changes.

### S/M overlap error
```text
Switch(config-if)#no ip address
Switch(config-if)#ip address 192.168.0.65 255.255.255.192
% 192.168.0.64 overlaps with Vlan120
```

.0.65 belonged to ENG Voice. .0.129 and .0.162 would also share one /26. S/M was finally assigned .1.64/26 and .1.128/26.

The command to remove an old IP could succeed before the new one failed. The interface then needed its correct address restored explicitly.

### Guest checks

A stale Guest lease first showed .0.195/27, then .1.195/26 after renewal. Wired and wireless clients later used the new subnet.

The collected wired ACL counters were:
```text
10 permit udp any eq bootpc any eq bootps (13 match(es))
20 permit icmp 192.168.1.192 0.0.0.63 host 192.168.1.193 echo (2 match(es))
30 deny ip any 10.0.0.0 0.255.255.255
40 deny ip any 172.16.0.0 0.15.255.255
50 deny ip any 192.168.0.0 0.0.255.255 (7 match(es))
60 permit ip 192.168.1.192 0.0.0.63 any (1 match(es))
```

Seven failed internal requests matched seven deny hits. DHCP remained allowed. Guest wireless later showed .1.196, gateway success, and server blocks.

### Static IP cleanup

IT printer moved to .1.30, HR printer to .0.221, and ENG printer to .0.62. S/M printer .1.126 was assigned in the plan but its test was not supplied.

ENG-Data was to be reduced to 58 users (.2–59). Its final pool screen was not collected. The last screen before that change still showed 60.

The serverPool .14 suggestion was not the final checked setting. The actual screen kept .1.6, /28, one user.

### Collected results

| Source | Result |
| :--- | :--- |
| IT .1.19/28 | Gateway 2/2, DHCP server 2/2, printer 4/4 |
| Web .1.3/28 | IT .1.19 replied 2/2 |
| S/M .1.67/26 | Gateway 2/2, DHCP server 3/3, Web 2/3 |
| Guest wired .1.195/26 | Gateway 2/2, servers blocked 0/4 and 0/3 |
| HR .0.195/27 | Gateway 2/2, DHCP 3/3, Web 2/2, printer 2/2 |
| ENG PC | Printer 3/3, DHCP server 2/2 |
| Guest wireless .1.196/26 | Gateway 2/2, both servers blocked 0/2 |

S/M's first Web timeout might have been ARP; it was not proven. Network address .1.192 was not a valid endpoint test. Typing errors in Ping/show commands were not network failures.

Recorded test commands:
```text
! IT PC
ipconfig /all
ping 192.168.1.17
ping 192.168.1.2
ping 192.168.1.30

! S/M PC
ipconfig /all
ping 192.168.1.65
ping 192.168.1.2
ping 192.168.1.3
ping 192.168.1.126

! HR PC
ipconfig /all
ping 192.168.0.193
ping 192.168.1.2
ping 192.168.1.3
ping 192.168.0.221

! ENG PC
ping 192.168.0.62
ping 192.168.1.2

! Guest wired / wireless: Gateway success, internal targets blocked
ipconfig /all
ping 192.168.1.193
ping 192.168.1.2
ping 192.168.1.3

! Core
show ip interface brief
show running-config
show access-lists
show interfaces trunk
```

Phone registration was still pending at this stage. Some pool, NIC-mask, printer, and final config evidence also remained open.

## 8. SSH on Core, IT, and HR

Core used .1.17, IT-Switch .1.28, and HR-Switch .0.220. The wired IT subnet .1.16/28 was the only allowed source.

Other access switches were deliberately left out because their setup would repeat the same process. Router SSH was not configured in this task.

Core configuration:
```text
enable
configure terminal
hostname Core-Switch
ip domain-name fakecompany.lab

username netadmin privilege 15 secret <LAB_PASSWORD>

ip access-list standard SSH-IT-ONLY
 permit 192.168.1.16 0.0.0.15
 deny any
exit

crypto key generate rsa
! When prompted for modulus size: 2048

ip ssh version 2

line vty 0 4
 login local
 transport input ssh
 access-class SSH-IT-ONLY in
 exec-timeout 5 0
exit

end
show ip ssh
show access-lists
```

IT configuration:
```text
enable
configure terminal

interface vlan 40
 ip address 192.168.1.28 255.255.255.240
 no shutdown
exit
ip default-gateway 192.168.1.17

ip domain-name fakecompany.lab
username netadmin privilege 15 secret <LAB_PASSWORD>

ip access-list standard SSH-IT-ONLY
 permit 192.168.1.16 0.0.0.15
 deny any
exit

crypto key generate rsa
! When prompted for modulus size: 2048
ip ssh version 2

line vty 0 15
 login local
 transport input ssh
 access-class SSH-IT-ONLY in
 exec-timeout 5 0
exit

end
```

HR configuration:
```text
enable
configure terminal
hostname HR-Switch

interface vlan 10
 ip address 192.168.0.220 255.255.255.224
 no shutdown
exit
ip default-gateway 192.168.0.193

ip domain-name fakecompany.lab
username netadmin privilege 15 secret <LAB_PASSWORD>

ip access-list standard SSH-IT-ONLY
 permit 192.168.1.16 0.0.0.15
 deny any
exit

crypto key generate rsa
! When prompted for modulus size: 2048
ip ssh version 2

line vty 0 15
 login local
 transport input ssh
 access-class SSH-IT-ONLY in
 exec-timeout 5 0
exit

end
```

The actual RSA key-size output was not collected. The core VTY evidence covered 0–4; any other existing range would need the same policy.

### SSH version check
```text
SSH Enabled - version 2.0
Authentication timeout: 120 secs; Authentication retries: 3
```

The activation log mentioned 1.99, but show ip ssh confirmed 2.0.

### Core authentication reset
```text
configure terminal
username netadmin privilege 15 secret <NEW_LAB_PASSWORD>
line vty 0 4
 login local
exit
end
```

The first connection reached Password prompts and closed. The exact original password/input issue was not confirmed.

A later refusal was traced to this ACL:
```text
Standard IP access list SSH-IT-ONLY
 10 permit 192.168.1.0 0.0.0.15
 20 deny any (10 match(es))
```

Corrected ACL:
```text
configure terminal
ip access-list standard SSH-IT-ONLY
 no permit 192.168.1.0 0.0.0.15
 no deny any
 permit 192.168.1.16 0.0.0.15
 deny any
exit
end
```

The old rule allowed .1.0–15, not IT clients .18–27. The permit was placed before deny any.

### IT authentication reset
```text
configure terminal
username netadmin privilege 15 secret <NEW_LAB_PASSWORD>
line vty 0 15
 login local
exit
end
```

IT-Switch then accepted the login. Each device keeps its own local account.

### SSH results

| Device | Result | ACL |
| :--- | :--- | :--- |
| Core | Core-Switch# | Permit 2 / deny 5 |
| IT | IT-Switch# | Permit 16 / deny 16 |
| HR | HR-Switch# | Permit 2 / deny 15 |

Counters count packets, not people or sessions. The client snippets did not include source IPs; roles follow the lab test steps. IT/HR show ip ssh, full final configs, and Telnet tests were not collected.

The command explanations covered enable, configure terminal, local accounts, privilege 15, VTY ranges, login local, SSH-only transport, access-class, timeout, exit, and end.

The HR question was resolved by separating Core .193 as gateway from HR-Switch .220 as its own management IP.

## 9. Edge routing, PAT, DNS, and phones

### Transit and routes

Core:
```text
configure terminal
interface gigabitEthernet0/1
 description TO-EDGE-ROUTER
 no switchport
 ip address 10.0.0.1 255.255.255.252
 no shutdown
exit
ip route 0.0.0.0 0.0.0.0 10.0.0.2
end
```

Edge:
```text
configure terminal
hostname Edge-Router
interface gigabitEthernet0/0
 description TO-CORE-SWITCH
 ip address 10.0.0.2 255.255.255.252
 no shutdown
exit
ip route 192.168.0.0 255.255.255.0 10.0.0.1
ip route 192.168.1.0 255.255.255.0 10.0.0.1

interface gigabitEthernet0/1
 description TO-EXTERNAL-WEB
 ip address 203.0.113.1 255.255.255.0
 no shutdown
exit
end
```

Edge→Core Ping was 4/5, with the first request missing. C/L entries and .0/24 and .1/24 static routes through 10.0.0.1 were supplied. IT and external connection tests were reported complete.

External-Web was .113.10/24 with gateway .113.1, HTTP On. It was directly connected to Edge Gi0/1. No ISP or real Google access was built.

### PAT
```text
configure terminal
ip access-list standard NAT-INTERNAL
 permit 192.168.0.0 0.0.0.255
 permit 192.168.1.0 0.0.0.255
exit
interface gigabitEthernet0/0
 ip nat inside
exit
interface gigabitEthernet0/1
 ip nat outside
exit
ip nat inside source list NAT-INTERNAL interface gigabitEthernet0/1 overload
end
```

At this stage, external HTTP and Guest blocking were reported complete, but the NAT output was still pending. Later Section 10 closed that evidence gap for IT/Staff.

### DNS

Internal DNS .1.2 had intranet.fakecompany.test → .1.3. External DNS 203.0.113.10 had www.external.test → 203.0.113.10.

Data pools pointed to internal DNS; Guest pointed to external DNS. Records were added under Services → DNS, and leases were refreshed.

The first internal hostname failed while Ping and IP HTTP worked. DNS Service was Off. Turning it On fixed the test. Internal names across four Data pools and Guest external names were reported working. Raw nslookup/screenshots were not collected.

### Voice setup

The current 2911 rejected telephony-service. A separate 2811 accepted it. The original two-phone sample was expanded to eight after the existing layout was checked.

Server switch:
```text
configure terminal
interface fastEthernet0/3
 description TO-VOICE-ROUTER
 switchport mode access
 switchport access vlan 50
 spanning-tree portfast
 no shutdown
exit
end
```

Voice router:
```text
configure terminal
hostname Voice-Router
interface fastEthernet0/0
 description TO-SERVER-SWITCH
 ip address 192.168.1.4 255.255.255.240
 no shutdown
exit
ip route 0.0.0.0 0.0.0.0 192.168.1.1

telephony-service
 max-ephones 8
 max-dn 8
 ip source-address 192.168.1.4 port 2000
 auto assign 1 to 8
exit
ephone-dn 1
 number 1001
exit
ephone-dn 2
 number 1002
exit
ephone-dn 3
 number 1003
exit
ephone-dn 4
 number 1004
exit
ephone-dn 5
 number 1005
exit
ephone-dn 6
 number 1006
exit
ephone-dn 7
 number 1007
exit
ephone-dn 8
 number 1008
exit
telephony-service
 create cnf-files
exit
end
```

All Voice pools used TFTP .1.4. Auto assignment gave 1001–1008 by registration order, not fixed department mapping.

The Configuring CM List issue had two findings: router IP .1.3 duplicated Web, and CME source-address was missing. The router was changed to .4, source-address added, files rebuilt, and a phone restarted.

Actual phone output:
```text
ephone-1 Mac:0060.3E43.7B36
REGISTERED in SCCP ver 12 and Server in ver 8
IP:192.168.0.226
button 1: dn 1 number 1001 CH1 IDLE
```

REGISTERED confirmed registration; IDLE meant no active call. HR–ENG calls, eight registrations, and S/M–IT calls were later reported complete.

### Evidence commands and scope decisions
```text
! Core / Edge
show ip interface brief
show ip route
! Edge, generate client traffic first
show ip nat translations
show ip nat statistics
! Voice-Router
show ephone
! Switches
show interfaces trunk
show access-lists
! Clients
ipconfig /all
nslookup intranet.fakecompany.test
! Guest
nslookup www.external.test
```

Staff Wi-Fi, Syslog, and backup were still pending at this point. Python automation was excluded. Real public Internet required a separate suitable environment, not a renamed simulated server.

Reference: [Cisco Community: Packet Tracer Internet access](https://community.cisco.com/t5/cisco-software-discussions/packet-tracer-accessing-internet/td-p/4722181).

## 10. Staff Wi-Fi, logs, final PAT evidence, and backup

### Staff wireless

VLAN 60 used 192.168.2.0/25, gateway .2.1, DNS .1.2. Staff-WiFi DHCP started .2.10 with 110 users (.10–119). The initial 100-user example was replaced by the final 110-user setting.

Staff-AP used IT Fa0/4, SSID FakeCompany-Staff, channel 1, WPA2-PSK/AES, and a separate lab key. It remained logically separate from IT Data.

Core:
```text
configure terminal
vlan 60
 name STAFF-WIFI
exit
interface vlan 60
 description STAFF-WIFI-GATEWAY
 ip address 192.168.2.1 255.255.255.128
 ip helper-address 192.168.1.2
 no shutdown
exit
interface fastEthernet0/4
 switchport trunk allowed vlan add 60
exit
end
```

IT:
```text
configure terminal
vlan 60
 name STAFF-WIFI
exit
interface gigabitEthernet0/1
 switchport trunk allowed vlan add 60
exit
interface fastEthernet0/4
 description TO-STAFF-AP
 switchport mode access
 switchport access vlan 60
 spanning-tree portfast
 no shutdown
exit
end
```

Edge:
```text
configure terminal
ip route 192.168.2.0 255.255.255.128 10.0.0.1
ip access-list standard NAT-INTERNAL
 permit 192.168.2.0 0.0.0.127
exit
end
```

The new subnet needed a return route and NAT-source addition. Trunk add preserved the existing IT VLANs.

Wireless0 .2.11 was supplied, with 2/2 replies from .2.1, .1.2, and 203.0.113.10. The full wireless mask/gateway/DNS output was cut off. Internal name access and SSH blocking were reported complete.

This was a sample-client test, not a real AP capacity test, 802.1X, or per-user authentication rollout.

### Central Syslog

Core:
```text
configure terminal
service timestamps log datetime msec
logging 192.168.1.2
logging trap debugging
logging on
end
show logging
```

informational and numeric 6 were rejected. debugging worked. It set a severity threshold, not separate debug tracing.

Core sending output:
```text
Syslog logging: enabled
Trap logging: level debugging
Logging to 192.168.1.2 (udp port 514 ... link up)
2 message lines logged
```

The server screen showed three messages from .1.1: two CONFIG_I and one LOGGINGHOST_STARTSTOP. A new configuration-mode event was received.

This confirmed event reception. It did not implement SNMP performance graphs, regular health checks, automatic alerts, or full command auditing. The simulator timestamps were not synchronized.

### Actual PAT entries
```text
Pro Inside global      Inside local        Outside local      Outside global
tcp 203.0.113.1:1027   192.168.2.11:1027   203.0.113.10:80    203.0.113.10:80
tcp 203.0.113.1:1031   192.168.1.19:1031   203.0.113.10:80    203.0.113.10:80

Total translations: 2 (0 static, 2 dynamic, 2 extended)
Outside Interfaces: GigabitEthernet0/1
Inside Interfaces: GigabitEthernet0/0
Hits: 20  Misses: 8
Expired translations: 6
```

Staff .2.11 and IT .1.19 shared 203.0.113.1 for HTTP to .113.10. Misses were not treated as lost packets. This was simulated external access, not public Internet.

### Backup and save
```text
enable
copy running-config startup-config
copy running-config tftp:
! Remote host: 192.168.1.2
! Destination filename: use device-specific name below
```

The server was .1.2, TFTP On. Phone TFTP .4 was a different service role.

| Device | File | Recorded state |
| :--- | :--- | :--- |
| Core | Core-Switch-final.cfg | Instructions given; individual result pending |
| Edge | Edge-Router-final.cfg | Instructions given; individual result pending |
| Voice | Voice-Router-final.cfg | Instructions given; individual result pending |
| IT | IT-Switch-final.cfg | Startup [OK] checked; TFTP completion reported |

HR/Server startup saves were requested. No restore test was run. The final completion report followed the IT backup, so it was not used to infer all other file copies.

The saved [final .pkt](../../fake-company-final.pkt) was later found in the repository. It was not reopened to check embedded backup files. A TFTP file inside the .pkt is a lab backup, not a separate host-loss-safe copy.

### Final scope

The lab covered VLANs, new addressing, DHCP/relay, static IP separation, routing/PAT, Guest isolation, sample SSH, DNS, phones, two Wi-Fi networks, central core logs, and manual backup practice.

Excluded or untested: real Internet, Windows AD, SSH on every switch, Python automation, external PRTG/Zabbix, SNMP dashboards/alerts, automatic backup, and restore testing.

Remaining evidence included ENG pool 58, some final NIC/printer/config details, individual backup files, and saved-device checks. No new feature was required to keep expanding this project.

## Screenshot history

The resource images include early HR wiring/VLANs, core links, the router addition, three DHCP stages, and the final topology. The [screenshot index](../../resources/readme.md) explains which values are historical.

---

[Back to main README](../../readme.md) · [Current guide](../guide.md) · [Current test results](../validation.md)

## 11. Backup and restore closeout

This update records new evidence after the earlier archive sections. Statements above about an untested restore describe the earlier state.

- The lab owner explicitly confirmed all eight phone registrations and interdepartment calls were tested successfully.
- The lab owner confirmed the backup task was complete. Full non-IT-switch transfer/file listings were not supplied.
- IT-Switch startup save returned [OK].
- Upload of IT-Switch-final.cfg to 192.168.1.2 returned [OK - 1979 bytes].
- The hostname was changed to RESTORE-TEST.
- copy tftp: running-config downloaded the same file, returned [OK - 1979 bytes], and changed the prompt back to IT-Switch#.
- VLAN 40 was 192.168.1.28 up/up. SSH-IT-ONLY retained the correct permit/deny entries.
- IT PC Ping returned 2/3 replies, with the first request timing out. SSH reached IT-Switch#.

This closes the sample restore test as **output checked**. It tests a running-config merge and one deliberately changed value, not a full rebuild or reboot recovery. The successful download directly confirms the IT backup file was readable on TFTP.

[Full restore evidence](../validation.md#it-switch-backup-and-restore-test) · [Restore steps](../guide.md)
