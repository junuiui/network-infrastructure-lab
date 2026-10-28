# Step-by-Step Rebuild Guide

[Back to main README](../readme.md) · [Address plan](addressing.md) · [Troubleshooting](troubleshooting.md) · [Test results](validation.md)

> Start with a blank Packet Tracer file. This guide builds the final design directly. It is not a script for changing the old addresses on an existing lab.
>
> The commands were put together from the lab record. The full guide was not rerun from a blank file during this documentation update. Use `?` if your model or Packet Tracer version accepts different commands.

## 0. Prepare the lab

| Item | Quantity / use |
| :--- | :--- |
| 3560-24PS | One core switch |
| 2960 | Six access switches: HR, ENG, S/M, IT, Server, Guest |
| 2911 | Edge routing and PAT |
| 2811 | Voice router with CME |
| Server-PT | DHCP/internal DNS/Syslog/TFTP, internal Web, external Web/DNS |
| 7960 phones | Two per department, eight total |
| PCs and printers | Sample clients for each department |
| AccessPoint-PT | Guest and Staff APs |
| Laptop-PT / WPC300N | Guest and Staff wireless clients |

Use sample devices rather than the full company count. Read [Architecture](architecture.md) and [Addressing](addressing.md) first.

CLI modes:

| Prompt | Meaning | How to enter |
| :--- | :--- | :--- |
| > | User EXEC | Open the CLI |
| # | Privileged EXEC | enable |
| (config)# | Global configuration | configure terminal |
| (config-if)# | Interface configuration | interface ... |

`exit` goes back one level. `end` returns to privileged EXEC. Passwords shown as `<LAB_PASSWORD>` are placeholders; enter your own lab-only value.

## 1. Place devices and connect cables

Follow the [port map](architecture.md).

- Core Fa0/1–6 connect to HR, ENG, S/M, IT, Server, and Guest Gi0/1.
- Core Gi0/1 connects to Edge Gi0/0.
- Edge Gi0/1 connects to External-Web FastEthernet0.
- Server Fa0/1 connects to DHCP, Fa0/2 to Web, Fa0/3 to Voice Fa0/0.
- Guest Fa0/3 connects to Guest-AP Port 0.
- IT Fa0/4 connects to Staff-AP Port 0.
- Department Fa0/1–2 connect to phone **Switch** ports. Connect PCs to the phone **PC** ports.
- Department Fa0/3 connects to the printer.

Use automatic cable selection if needed. Give each phone power through supported PoE or its Physical-tab power adapter.

![HR phone and PC wiring](../resources/1-HR-structure.png)

*The phone and PC share a cable path. Their VLAN settings separate the traffic.*

## 2. Configure the core VLANs, trunks, and gateways

- **VLAN:** A separate Layer 2 broadcast area.
- **Trunk:** One link that carries several VLANs.
- **SVI:** A VLAN interface with an IP; it is the client gateway here.
- **ip routing:** Lets the core route between VLANs.
- **ip helper-address:** Relays client DHCP requests to the central server.

On Core:
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

This guide sets explicit trunks and allowed VLAN lists. The older lab used negotiated trunks on Fa0/1–3.

Check:
```text
show vlan brief
show ip interface brief
show interfaces trunk
```

An SVI may stay down until its VLAN has an active forwarding connection. Check again after adding the access switches.

![HR VLAN list](../resources/1-HR-VLAN.png)

*Example: Data VLAN 10 and Voice VLAN 110 are separate entries.*

## 3. Configure the access switches

Phone ports need both Data and Voice VLANs. A dedicated printer port only needs the Data VLAN. The Staff AP uses a separate access port in VLAN 60.

### 3.1 HR-Switch
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

### 3.2 ENG-Switch
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

### 3.3 S/M-Switch
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

### 3.4 IT-Switch
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

### 3.5 Server-Switch
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

### 3.6 Guest-Switch
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

Check `show vlan brief` and `show interfaces trunk` on each switch. For a phone port, use:
```text
show interfaces fastEthernet0/1 switchport
```

The access VLAN should match the department Data VLAN, and Voice VLAN should match its phone network.

## 4. Set static server and printer addresses

For servers, open **Desktop → IP Configuration → Static**.

| Device | IP | Mask | Gateway |
| :--- | :--- | :--- | :--- |
| DHCP-Server | 192.168.1.2 | 255.255.255.240 | 192.168.1.1 |
| Web-Server | 192.168.1.3 | 255.255.255.240 | 192.168.1.1 |
| HR printer | 192.168.0.221 | 255.255.255.224 | 192.168.0.193 |
| ENG printer | 192.168.0.62 | 255.255.255.192 | 192.168.0.1 |
| S/M printer | 192.168.1.126 | 255.255.255.192 | 192.168.1.65 |
| IT printer | 192.168.1.30 | 255.255.255.240 | 192.168.1.17 |

Use the printer Config/Ethernet and Settings pages for its IP, mask, and gateway.

Reserve .1.6 for serverPool if it is kept. Reserve .1.28 for IT-Switch and .0.220 for HR-Switch. Do not give those addresses to other devices.

On Web-Server, turn **Services → HTTP → On**. Test IP-based HTTP before testing names.

## 5. Add the central DHCP pools

On DHCP-Server, open **Services → DHCP → On**. Use the [final pool table](addressing.md).

- Click **Add** for a new pool or **Save** for an existing pool.
- Staff Data and Staff-WiFi DNS: **192.168.1.2**.
- Guest DNS: **203.0.113.10**.
- All four Voice pools TFTP: **192.168.1.4**.
- ENG-Data Max Users: **58**, giving .0.2–59.
- Staff-WiFi: start .2.10, /25, gateway .2.1, **110 users**.
- Remove the default serverPool if possible. To match this lab when it must remain, use start .1.6, /28, gateway .1.1, Max Users 1.

On each PC, choose **Desktop → IP Configuration → DHCP**. To refresh an old lease, select Static, then DHCP.

Check IP, mask, gateway, and server using `ipconfig /all`. Ping the gateway and 192.168.1.2 from a staff client.

![DHCP configuration screen during the address changes](../resources/dhcp-server-pool-fixed.png)

*Use this image to locate the fields. Its values are from the middle of the migration. Use the final table, not the screenshot, for the new build.*

## 6. Apply the Guest ACL

The core must allow DHCP before blocking private destinations. Apply the ACL inbound on Guest VLAN 99.

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

From Guest, Ping .1.193 should pass. Pings to .1.2 and .1.3 should fail. Check `show access-lists` for deny matches.

A failed Ping alone does not prove an ACL block. Check the working gateway and counters too.

## 7. Set up routing and the external test server

Both directions need a route. Core needs a path to the edge, and the edge needs return paths to internal networks.

### Core
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

`no switchport` turns Gi0/1 into a routed port. The /30 connects the two devices. The default route handles destinations the core does not know more specifically.

### Edge
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

The return routes point to the core. They do not change client masks.

Set External-Web to **203.0.113.10**, mask **255.255.255.0**, gateway **203.0.113.1**. Turn **Services → HTTP → On**.

Check:

- Edge: Ping 10.0.0.1 and 203.0.113.10.
- IT PC: Ping 10.0.0.2.
- Edge/Core: `show ip route`.

C means connected, L means the local interface IP, and S means static route. Guest Ping to 10.0.0.2 is expected to be blocked.

## 8. Add PAT

PAT lets inside clients share the edge's outside address. The ACL selects sources for translation.

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

Open `http://203.0.113.10` from an IT PC and later from Staff Wi-Fi. Immediately check:
```text
show ip nat translations
show ip nat statistics
```

Inside local should show client addresses. Inside global should show **203.0.113.1**. Check the table even if the website opens; web access alone does not prove translation.

## 9. Set up internal and external DNS

On each server, open **Services → DNS → On**. Add an A record and click **Add**.

| Server | Name | Address |
| :--- | :--- | :--- |
| DHCP-Server, 192.168.1.2 | intranet.fakecompany.test | 192.168.1.3 |
| External-Web, 203.0.113.10 | www.external.test | 203.0.113.10 |

Do not put `http://` in the DNS record name. Check that the record is in the list.

Renew leases after changing the DHCP DNS field. On a staff PC:
```text
nslookup intranet.fakecompany.test
```

Then open `http://intranet.fakecompany.test`. On Guest, look up `www.external.test` and open its HTTP page.

If IP-based HTTP works but the name fails, check the client DNS address, DNS Service On, and record. This lab's first failure was DNS Service Off.

## 10. Configure SSH on sample devices

### 10.1 Add management IPs

Core already has **192.168.1.17** on VLAN 40.

IT-Switch:
```text
configure terminal
interface vlan 40
 ip address 192.168.1.28 255.255.255.240
 no shutdown
exit
ip default-gateway 192.168.1.17
end
```

HR-Switch:
```text
configure terminal
interface vlan 10
 ip address 192.168.0.220 255.255.255.224
 no shutdown
exit
ip default-gateway 192.168.0.193
end
```

These are the Layer 2 switches' own addresses. Their default gateways allow replies to admins in another subnet. Client gateways stay on the core.

### 10.2 Add the local account, RSA key, and ACL

On Core, IT, and HR, after setting the hostname:
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

At the RSA size prompt, enter **2048**. Check existing keys before replacing them on an old lab. This guide assumes new devices.

On IT and HR:
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

On Core:
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

If the core has another VTY range, apply the same policy there.

- `login local` checks this device's local account.
- `transport input ssh` limits these lines to SSH.
- `access-class` checks the source subnet.
- `exec-timeout 5 0` closes idle sessions after five minutes.
- VTY 0–15 selects 16 remote lines, not physical ports.
- The permit must be **192.168.1.16/28**, not .1.0/28.

Check:
```text
show ip ssh
show access-lists
```

From an IT PC, run:
```text
ssh -l netadmin 192.168.1.17
ssh -l netadmin 192.168.1.28
ssh -l netadmin 192.168.0.220
```

Run one session at a time and exit before the next. HR/S/M/Staff clients should be blocked. Password characters are not shown while typing. Do not publish passwords or hashes in the test record.

## 11. Add Guest and Staff Wi-Fi

Connect Guest-AP to Guest Fa0/3 and Staff-AP to IT Fa0/4. Those access ports and their uplinks were set in Steps 2–3.

| Setting | Guest | Staff |
| :--- | :--- | :--- |
| SSID | FakeCompany-Guest | FakeCompany-Staff |
| Channel | 6 | 1 |
| Authentication | WPA2-PSK | WPA2-PSK |
| Encryption | AES | AES |
| Password | Lab-only value | Different lab-only value |

On AccessPoint-PT, turn wired **Port 0** and wireless **Port 1 On**.

On each laptop:

1. Turn power Off in Physical.
2. Replace the wired module with WPC300N.
3. Turn power On.
4. Open **Desktop → PC Wireless**, refresh, choose the SSID, and enter the password.
5. Choose **IP Configuration → Wireless0 → DHCP**.

For adding Staff VLAN 60 to an existing lab, use `switchport trunk allowed vlan add 60` at **both** Core Fa0/4 and IT Gi0/1. It keeps the older allowed VLANs.

| Check | Guest | Staff |
| :--- | :--- | :--- |
| Address | .1.194–254 /26 | .2.10–119 /25 |
| Gateway | .1.193 | .2.1 |
| DNS | 203.0.113.10 | 192.168.1.2 |
| Internal server Ping | Blocked | Pass |
| External HTTP | Pass | Pass |
| Core SSH | Blocked | Blocked |

Staff must also open the internal website by name. Collect the full Wireless0 settings.

![Staff and Guest AP placement](../resources/final.png)

*Staff and Guest use separate APs and VLANs. The Staff AP uses a spare IT switch port.*

## 12. Set up CME and eight phones

Use the separate 2811. Confirm `telephony-service ?` returns `<cr>`.

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

| Command | Why |
| :--- | :--- |
| max-ephones / max-dn | Set phone and extension limits |
| ip source-address ... port 2000 | Set the SCCP registration address |
| ephone-dn / number | Create each extension |
| auto assign | Assign entries in registration order |
| create cnf-files | Make files for phones to download |

Check **TFTP Server 192.168.1.4** in all four Voice pools. Restart phones to get the new DHCP/config information.

```text
show ephone
```

Check eight REGISTERED entries and the correct Voice subnet. Use the number displayed on the destination phone, call from another department, pick up, confirm the connection, and hang up. Test HR–ENG and S/M–IT.

If a phone stays at Configuring CM List, check for duplicate .1.3, missing CME source-address, and the DHCP TFTP value.

## 13. Collect core Syslog events

On DHCP-Server, open **Services → SYSLOG → On**.

On Core:
```text
configure terminal
service timestamps log datetime msec
logging 192.168.1.2
logging trap debugging
logging on
end
show logging
```

In this environment, `informational` and numeric `6` were rejected. `debugging` worked. It sets level 7 and includes levels 0–7; it does not turn on separate debug commands.

Create a configuration event:
```text
configure terminal
end
```

Check the server for a new CONFIG_I message. The sending config alone is not proof of reception.

This is event collection, not a performance dashboard or automatic alert system. The simulator clock may differ from the real date.

## 14. Save and back up

On DHCP-Server, open **Services → TFTP → On**.

On every changed switch/router:
```text
copy running-config startup-config
```

On sample devices:
```text
copy running-config tftp:
```

Enter remote host **192.168.1.2**, then the matching file name.

| Device | File name |
| :--- | :--- |
| Core | Core-Switch-final.cfg |
| Edge | Edge-Router-final.cfg |
| Voice | Voice-Router-final.cfg |
| IT | IT-Switch-final.cfg |

Check the transfer message and TFTP server file list. Startup save and remote backup are separate actions.

Use **File → Save As** to save your new .pkt. The repository's saved final file is [fake-company-final.pkt](../fake-company-final.pkt). Keep a new build under its own name.

A restore test is not part of this procedure. If you add one, use a copy and document the result.

## 15. Final checks

1. Check sample client IP, mask, gateway, DNS, and no printer overlap.
2. Staff internal HTTP passes. Guest external HTTP passes and internal access is blocked.
3. IT SSH passes. Non-IT SSH is blocked, with ACL matches.
4. Edge PAT entries show wired IT and Staff Wi-Fi.
5. Eight phones register and calls connect between departments.
6. Syslog server receives a new event.
7. TFTP files, startup configs, and final .pkt are saved.
8. Record the actual output using [the test page](validation.md).

These are expected results for a new build. They do not turn uncollected results from the original lab into confirmed evidence.

---

[Back to main README](../readme.md) · [Architecture](architecture.md) · [Troubleshooting](troubleshooting.md) · [Test results](validation.md)
