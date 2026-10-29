# How To Build the Lab

[Back to README](readme.md) · [Problems and fixes](ProblemSolving.md)

Use a blank Packet Tracer file. Follow the sections in order. Replace `<LAB_PASSWORD>` with your own lab-only password. Do not paste the angle brackets into the command.

## 1. Add the devices

| Device | Model / count |
| :--- | :--- |
| Core-Switch | One 3560-24PS |
| HR-Switch, ENG-Switch, SM-Switch, IT-Switch | Four 2960 switches |
| Server-Switch, Guest-Switch | Two 2960 switches |
| Edge-Router | One 2911 |
| Voice-Router | One 2811 |
| DHCP-Server, Web-Server, External-Web | Three Server-PT devices |
| Department clients | Two 7960 phones and sample PCs per department |
| Printers | One sample printer per department |
| Guest-AP, Staff-AP | Two AccessPoint-PT devices |
| Guest-Laptop-1, Staff-Laptop-1 | Two Laptop-PT devices with WPC300N |

## 2. Connect the devices

| Device / port | Other end |
| :--- | :--- |
| Core Fa0/1 | HR Gi0/1 |
| Core Fa0/2 | ENG Gi0/1 |
| Core Fa0/3 | S/M Gi0/1 |
| Core Fa0/4 | IT Gi0/1 |
| Core Fa0/5 | Server Gi0/1 |
| Core Fa0/6 | Guest Gi0/1 |
| Core Gi0/1 | Edge Gi0/0 |
| Edge Gi0/1 | External-Web FastEthernet0 |
| Server Fa0/1 | DHCP-Server FastEthernet0 |
| Server Fa0/2 | Web-Server FastEthernet0 |
| Server Fa0/3 | Voice-Router Fa0/0 |
| Department Fa0/1–2 | Phone Switch ports |
| Phone PC ports | Department PCs |
| Department Fa0/3 | Department printer |
| Guest Fa0/1–2 | Wired Guest client ports |
| Guest Fa0/3 | Guest-AP Port 0 |
| IT Fa0/4 | Staff-AP Port 0 |

Use automatic cable selection. Supply phone power through a supported PoE port or a Physical-tab power adapter.

![Department phone and PC connections](resources/1-HR-structure.png)

## 3. Set the core VLANs, trunks, and SVIs

On **Core-Switch → CLI**:

```text
enable
configure terminal
hostname Core-Switch
ip routing

vlan 10
 name HR-DATA
exit
vlan 110
 name HR-VOICE
exit
vlan 20
 name ENG-DATA
exit
vlan 120
 name ENG-VOICE
exit
vlan 30
 name SM-DATA
exit
vlan 130
 name SM-VOICE
exit
vlan 40
 name IT-DATA
exit
vlan 140
 name IT-VOICE
exit
vlan 50
 name SERVER
exit
vlan 99
 name GUEST
exit
vlan 60
 name STAFF-WIFI
exit

interface fastEthernet0/1
 description TO-HR-SWITCH
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,110
 no shutdown
exit
interface fastEthernet0/2
 description TO-ENG-SWITCH
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 20,120
 no shutdown
exit
interface fastEthernet0/3
 description TO-SM-SWITCH
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 30,130
 no shutdown
exit
interface fastEthernet0/4
 description TO-IT-SWITCH
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 40,60,140
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

interface vlan 10
 ip address 192.168.0.193 255.255.255.224
 ip helper-address 192.168.1.2
 no shutdown
exit
interface vlan 110
 ip address 192.168.0.225 255.255.255.224
 ip helper-address 192.168.1.2
 no shutdown
exit
interface vlan 20
 ip address 192.168.0.1 255.255.255.192
 ip helper-address 192.168.1.2
 no shutdown
exit
interface vlan 120
 ip address 192.168.0.65 255.255.255.192
 ip helper-address 192.168.1.2
 no shutdown
exit
interface vlan 30
 ip address 192.168.1.65 255.255.255.192
 ip helper-address 192.168.1.2
 no shutdown
exit
interface vlan 130
 ip address 192.168.1.129 255.255.255.192
 ip helper-address 192.168.1.2
 no shutdown
exit
interface vlan 40
 ip address 192.168.1.17 255.255.255.240
 ip helper-address 192.168.1.2
 no shutdown
exit
interface vlan 140
 ip address 192.168.1.33 255.255.255.240
 ip helper-address 192.168.1.2
 no shutdown
exit
interface vlan 50
 ip address 192.168.1.1 255.255.255.240
 no shutdown
exit
interface vlan 99
 ip address 192.168.1.193 255.255.255.192
 ip helper-address 192.168.1.2
 no shutdown
exit
interface vlan 60
 ip address 192.168.2.1 255.255.255.128
 ip helper-address 192.168.1.2
 no shutdown
exit
end
```

Check:

```text
show vlan brief
show ip interface brief
show interfaces trunk
```

After access links are active, check the required SVIs are up/up.

![HR Data and Voice VLAN entries](resources/1-HR-VLAN.png)

*HR switch VLAN list.*

### Final gateway table

| Network | VLAN | Subnet | Gateway |
| :--- | ---: | :--- | :--- |
| HR Data | 10 | 192.168.0.192/27 | 192.168.0.193 |
| HR Voice | 110 | 192.168.0.224/27 | 192.168.0.225 |
| ENG Data | 20 | 192.168.0.0/26 | 192.168.0.1 |
| ENG Voice | 120 | 192.168.0.64/26 | 192.168.0.65 |
| S/M Data | 30 | 192.168.1.64/26 | 192.168.1.65 |
| S/M Voice | 130 | 192.168.1.128/26 | 192.168.1.129 |
| IT Data | 40 | 192.168.1.16/28 | 192.168.1.17 |
| IT Voice | 140 | 192.168.1.32/28 | 192.168.1.33 |
| Server | 50 | 192.168.1.0/28 | 192.168.1.1 |
| Guest | 99 | 192.168.1.192/26 | 192.168.1.193 |
| Staff Wi-Fi | 60 | 192.168.2.0/25 | 192.168.2.1 |

## 4. Set the access switches

### HR-Switch
```text
enable
configure terminal
hostname HR-Switch
vlan 10
exit
vlan 110
exit
interface range fastEthernet0/1 - 2
 switchport mode access
 switchport access vlan 10
 switchport voice vlan 110
 spanning-tree portfast
 no shutdown
exit
interface fastEthernet0/3
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 no shutdown
exit
interface gigabitEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan 10,110
 no shutdown
exit
end
```

### ENG-Switch
```text
enable
configure terminal
hostname ENG-Switch
vlan 20
exit
vlan 120
exit
interface range fastEthernet0/1 - 2
 switchport mode access
 switchport access vlan 20
 switchport voice vlan 120
 spanning-tree portfast
 no shutdown
exit
interface fastEthernet0/3
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast
 no shutdown
exit
interface gigabitEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan 20,120
 no shutdown
exit
end
```

### SM-Switch
```text
enable
configure terminal
hostname SM-Switch
vlan 30
exit
vlan 130
exit
interface range fastEthernet0/1 - 2
 switchport mode access
 switchport access vlan 30
 switchport voice vlan 130
 spanning-tree portfast
 no shutdown
exit
interface fastEthernet0/3
 switchport mode access
 switchport access vlan 30
 spanning-tree portfast
 no shutdown
exit
interface gigabitEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan 30,130
 no shutdown
exit
end
```

### IT-Switch
```text
enable
configure terminal
hostname IT-Switch
vlan 40
exit
vlan 140
exit
vlan 60
exit
interface range fastEthernet0/1 - 2
 switchport mode access
 switchport access vlan 40
 switchport voice vlan 140
 spanning-tree portfast
 no shutdown
exit
interface fastEthernet0/3
 switchport mode access
 switchport access vlan 40
 spanning-tree portfast
 no shutdown
exit
interface gigabitEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan 40,60,140
 no shutdown
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

### Server-Switch
```text
enable
configure terminal
hostname Server-Switch
vlan 50
exit
interface range fastEthernet0/1 - 3
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
```

### Guest-Switch
```text
enable
configure terminal
hostname Guest-Switch
vlan 99
exit
interface range fastEthernet0/1 - 3
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
```

On each switch, run `show vlan brief` and `show interfaces trunk`. On a department phone port, also run:

```text
show interfaces fastEthernet0/1 switchport
```

Check the department Data and Voice VLAN numbers.

## 5. Set static addresses

On Server-PT, open **Desktop → IP Configuration → Static**. On a printer, use its **Config → Ethernet** and **Settings** fields.

| Device | IP | Mask | Gateway |
| :--- | :--- | :--- | :--- |
| DHCP-Server | 192.168.1.2 | 255.255.255.240 | 192.168.1.1 |
| Web-Server | 192.168.1.3 | 255.255.255.240 | 192.168.1.1 |
| HR printer | 192.168.0.221 | 255.255.255.224 | 192.168.0.193 |
| ENG printer | 192.168.0.62 | 255.255.255.192 | 192.168.0.1 |
| S/M printer | 192.168.1.126 | 255.255.255.192 | 192.168.1.65 |
| IT printer | 192.168.1.30 | 255.255.255.240 | 192.168.1.17 |

Leave 192.168.1.6 for serverPool. Do not assign 192.168.1.28 or 192.168.0.220 to clients.

On Web-Server, turn **Services → HTTP → On**.

## 6. Add DHCP pools

On **DHCP-Server → Services → DHCP**, turn the service **On**. Enter each row and click **Add**. For an existing pool, edit and click **Save**.

| Pool | Gateway | Start IP | Mask | Max users | Lease range |
| :--- | :--- | :--- | :--- | ---: | :--- |
| ENG-Data | 192.168.0.1 | 192.168.0.2 | 255.255.255.192 | 58 | 192.168.0.2–59 |
| ENG-Voice | 192.168.0.65 | 192.168.0.66 | 255.255.255.192 | 60 | 192.168.0.66–125 |
| S/M-Data | 192.168.1.65 | 192.168.1.66 | 255.255.255.192 | 60 | 192.168.1.66–125 |
| S/M-Voice | 192.168.1.129 | 192.168.1.130 | 255.255.255.192 | 61 | 192.168.1.130–190 |
| Guests | 192.168.1.193 | 192.168.1.194 | 255.255.255.192 | 61 | 192.168.1.194–254 |
| HR-Data | 192.168.0.193 | 192.168.0.194 | 255.255.255.224 | 26 | 192.168.0.194–219 |
| HR-Voice | 192.168.0.225 | 192.168.0.226 | 255.255.255.224 | 29 | 192.168.0.226–254 |
| IT-Data | 192.168.1.17 | 192.168.1.18 | 255.255.255.240 | 10 | 192.168.1.18–27 |
| IT-Voice | 192.168.1.33 | 192.168.1.34 | 255.255.255.240 | 13 | 192.168.1.34–46 |
| Staff-WiFi | 192.168.2.1 | 192.168.2.10 | 255.255.255.128 | 110 | 192.168.2.10–119 |
| serverPool | 192.168.1.1 | 192.168.1.6 | 255.255.255.240 | 1 | 192.168.1.6 only |

Set these additional fields:

| Pools | DNS Server | TFTP Server |
| :--- | :--- | :--- |
| HR/ENG/S/M/IT Data and Staff-WiFi | 192.168.1.2 | 0.0.0.0 |
| Guests | 203.0.113.10 | 0.0.0.0 |
| HR/ENG/S/M/IT Voice | 0.0.0.0 | 192.168.1.4 |
| serverPool | 0.0.0.0 | 0.0.0.0 |

If the default serverPool can be removed, remove it. Otherwise use its single-address settings above.

On each PC, select **Desktop → IP Configuration → DHCP**. To renew an existing lease, select **Static**, then **DHCP**.

Run:

```text
ipconfig /all
```

Check the address, mask, gateway, and DNS against the tables. From a staff client, Ping its gateway and 192.168.1.2.

## 7. Apply the Guest ACL

On **Core-Switch**:

```text
configure terminal
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
exit
end
```

From Guest:

```text
ping 192.168.1.193
ping 192.168.1.2
ping 192.168.1.3
```

Expect the gateway to reply and both internal targets to be blocked. Run `show access-lists` on Core and check the matching counters.

## 8. Configure the edge link and routes

### Core-Switch
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

### Edge-Router
```text
enable
configure terminal
hostname Edge-Router
interface gigabitEthernet0/0
 description TO-CORE-SWITCH
 ip address 10.0.0.2 255.255.255.252
 no shutdown
exit
interface gigabitEthernet0/1
 description TO-EXTERNAL-WEB
 ip address 203.0.113.1 255.255.255.0
 no shutdown
exit
ip route 192.168.0.0 255.255.255.0 10.0.0.1
ip route 192.168.1.0 255.255.255.0 10.0.0.1
ip route 192.168.2.0 255.255.255.128 10.0.0.1
end
```

### External-Web

Open **Desktop → IP Configuration → Static**:

| Field | Value |
| :--- | :--- |
| IP | 203.0.113.10 |
| Mask | 255.255.255.0 |
| Gateway | 203.0.113.1 |

Turn **Services → HTTP → On**.

Check from Edge:

```text
ping 10.0.0.1
ping 203.0.113.10
show ip route
```

Check from an IT PC:

```text
ping 10.0.0.2
```

## 9. Configure PAT

On **Edge-Router**:

```text
configure terminal
ip access-list standard NAT-INTERNAL
 permit 192.168.0.0 0.0.0.255
 permit 192.168.1.0 0.0.0.255
 permit 192.168.2.0 0.0.0.127
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

From an IT PC, open **Desktop → Web Browser**:

```text
http://203.0.113.10
```

Immediately run on Edge:

```text
show ip nat translations
show ip nat statistics
```

Check the client address under Inside local and 203.0.113.1 under Inside global. Repeat after the Staff laptop is connected.

## 10. Add DNS records

On each server, open **Services → DNS**, select **On**, enter an **A Record**, and click **Add**.

| Server | Name | Address |
| :--- | :--- | :--- |
| DHCP-Server | intranet.fakecompany.test | 192.168.1.3 |
| External-Web | www.external.test | 203.0.113.10 |

Enter the record name without `http://`. Check it appears in the list.

Renew the client DHCP lease. On staff:

```text
nslookup intranet.fakecompany.test
```

Open `http://intranet.fakecompany.test`. On Guest, query `www.external.test` and open `http://www.external.test`.

## 11. Configure SSH

### IT-Switch management address
```text
configure terminal
interface vlan 40
 ip address 192.168.1.28 255.255.255.240
 no shutdown
exit
ip default-gateway 192.168.1.17
end
```

### HR-Switch management address
```text
configure terminal
interface vlan 10
 ip address 192.168.0.220 255.255.255.224
 no shutdown
exit
ip default-gateway 192.168.0.193
end
```

### Account, ACL, and RSA keys

On **Core, IT, and HR**, run:

```text
configure terminal
ip domain-name fakecompany.lab
username netadmin privilege 15 secret <LAB_PASSWORD>
ip access-list standard SSH-IT-ONLY
 permit 192.168.1.16 0.0.0.15
 deny any
exit
crypto key generate rsa
```

Enter **2048** at the RSA size prompt.

On **IT and HR**:

```text
ip ssh version 2
line vty 0 15
 login local
 transport input ssh
 access-class SSH-IT-ONLY in
 exec-timeout 5 0
exit
end
```

On **Core**:

```text
ip ssh version 2
line vty 0 4
 login local
 transport input ssh
 access-class SSH-IT-ONLY in
 exec-timeout 5 0
exit
end
```

If another VTY range appears in the core configuration, apply the same settings to it.

Check:

```text
show ip ssh
show access-lists
```

From an IT PC, open one session at a time and exit before the next:

```text
ssh -l netadmin 192.168.1.17
ssh -l netadmin 192.168.1.28
ssh -l netadmin 192.168.0.220
```

From HR/S/M/Staff Wi-Fi, check that the same management connections are blocked.

## 12. Connect Staff and Guest Wi-Fi

On each **AccessPoint-PT → Config**, turn wired **Port 0** and wireless **Port 1 On**.

| Field | Guest | Staff |
| :--- | :--- | :--- |
| SSID | FakeCompany-Guest | FakeCompany-Staff |
| Channel | 6 | 1 |
| Authentication | WPA2-PSK | WPA2-PSK |
| Encryption | AES | AES |
| Pass Phrase | Lab-only key | A different lab-only key |

On each Laptop:

1. Open **Physical** and turn power Off.
2. Replace the wired module with **WPC300N**.
3. Turn power On.
4. Open **Desktop → PC Wireless → Connect → Refresh**.
5. Select the SSID, connect, and enter the matching key.
6. Open **IP Configuration → Wireless0 → DHCP**.

| Check | Guest | Staff |
| :--- | :--- | :--- |
| IP | 192.168.1.194–254 | 192.168.2.10–119 |
| Mask | 255.255.255.192 | 255.255.255.128 |
| Gateway | 192.168.1.193 | 192.168.2.1 |
| DNS | 203.0.113.10 | 192.168.1.2 |
| Internal server access | Blocked | Pass |
| External test HTTP | Pass | Pass |
| Core SSH | Blocked | Blocked |

## 13. Configure IP phones

On **Voice-Router**, first check:

```text
enable
configure terminal
telephony-service ?
end
```

Check that `<cr>` appears. Return to the privileged prompt and enter the configuration below.

```text
enable
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

Check **TFTP Server = 192.168.1.4** in all four Voice DHCP pools. Restart the phones.

On Voice-Router:

```text
show ephone
```

Check eight REGISTERED entries and the correct Voice addresses. Read the extension on each phone. From HR, dial ENG and pick up on ENG. Confirm connection and hang up. Repeat between S/M and IT.

## 14. Configure Syslog

On **DHCP-Server → Services → SYSLOG**, turn the service **On**.

On **Core**:

```text
configure terminal
service timestamps log datetime msec
logging 192.168.1.2
logging trap debugging
logging on
end
show logging
```

Create a test event:

```text
configure terminal
end
```

Open the server SYSLOG list and check a new CONFIG_I entry.

## 15. Save and back up

On **DHCP-Server → Services → TFTP**, turn the service **On**.

On each changed switch and router:

```text
copy running-config startup-config
```

On each device being backed up:

```text
copy running-config tftp:
```

Enter remote host **192.168.1.2** and the matching file name:

| Device | File name |
| :--- | :--- |
| Core | Core-Switch-final.cfg |
| Edge | Edge-Router-final.cfg |
| Voice | Voice-Router-final.cfg |
| IT | IT-Switch-final.cfg |

Check the successful transfer and the server file list.

Save the Packet Tracer file. The existing project files are [fake-company-final.pkt](fake-company-final.pkt) and [fake-company-restore-test.pkt](fake-company-restore-test.pkt).

## 16. Test an IT-Switch restore

1. Use **File → Save As** to make a separate test copy.
2. Confirm **IT-Switch-final.cfg** is on the TFTP server and contains hostname **IT-Switch**.
3. Use the IT switch's direct console.
4. Do not overwrite the backup after the temporary change.

Run:

```text
enable
configure terminal
hostname RESTORE-TEST
end
copy tftp: running-config
```

Enter:

| Prompt | Value |
| :--- | :--- |
| Remote host | 192.168.1.2 |
| Source filename | IT-Switch-final.cfg |
| Destination filename | Press Enter for running-config |

Check that the prompt returns to **IT-Switch#**. Run:

```text
show ip interface brief
show access-lists
```

Check VLAN 40 is **192.168.1.28 up/up** and the ACL has **permit 192.168.1.16 0.0.0.15** before **deny any**.

From an IT PC:

```text
ping 192.168.1.28
ssh -l netadmin 192.168.1.28
```

Check replies and an SSH login to **IT-Switch#**. Keep the transfer output and test results. Save the desired startup configuration and test .pkt after checking them.

## 17. Final checks

| Area | Check |
| :--- | :--- |
| Client addressing | Check IP, mask, gateway, and DNS |
| Printers | Check static IPs are outside DHCP ranges and Ping the configured addresses |
| Guest | Check gateway success, internal block, and external-name HTTP |
| Staff Wi-Fi | Check internal/external access and denied management SSH |
| SSH | Check IT login and non-IT block on Core, IT, and HR |
| PAT | Open external HTTP and inspect translations/statistics |
| Phones | Check registration and calls between departments |
| Syslog | Check a new event on the server |
| Backup | Check transfer success and saved files |
| Restore | Check hostname, management SVI, ACL, and SSH |
| Save | Save changed startup configs and the .pkt |

[Back to README](readme.md) · [Problems and fixes](ProblemSolving.md)
