# Staff and Guest Wi-Fi

[Back to main README](../readme.md) · [Rebuild guide](guide.md) · [Test results](validation.md)

## Two networks, two purposes

| Setting | Guest | Staff |
| :--- | :--- | :--- |
| AP / client | Guest-AP / Guest-Laptop-1 | Staff-AP / Staff-Laptop-1 |
| SSID | FakeCompany-Guest | FakeCompany-Staff |
| Switch port | Guest Fa0/3 | IT Fa0/4 |
| VLAN | 99 | 60 |
| Network | 192.168.1.192/26 | 192.168.2.0/25 |
| Gateway | 192.168.1.193 | 192.168.2.1 |
| DNS | 203.0.113.10 | 192.168.1.2 |
| DHCP range | 192.168.1.194–254 | 192.168.2.10–119 |
| Internal services | Blocked | Allowed |
| Switch SSH | Blocked | Blocked |
| Security / channel | WPA2-PSK, AES / 6 | WPA2-PSK, AES / 1 |

![Final layout showing Staff AP on the IT switch and Guest AP on the Guest switch](../resources/final.png)

*The APs use different VLANs. Connecting Staff-AP to the IT switch does not put staff wireless clients in IT Data VLAN 40.*

## Switch setup

AP connections are access ports because each AP carries one VLAN. The uplink to the core is a trunk.

Guest AP port:
```text
interface fastEthernet0/3
 description TO-GUEST-AP
 switchport mode access
 switchport access vlan 99
 spanning-tree portfast
 no shutdown
```

Staff VLAN and AP port on IT-Switch:
```text
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
```

Also add VLAN 60 on Core Fa0/4:
```text
interface fastEthernet0/4
 switchport trunk allowed vlan add 60
```

`add 60` keeps the existing allowed VLANs. Replacing the whole list with 60 would remove IT Data/Voice from that trunk.

## Staff gateway, DHCP relay, and return path

Core:
```text
vlan 60
 name STAFF-WIFI
exit
interface vlan 60
 ip address 192.168.2.1 255.255.255.128
 ip helper-address 192.168.1.2
 no shutdown
```

DHCP-Server: add Staff-WiFi with gateway .2.1, start .2.10, mask 255.255.255.128, DNS .1.2, and **110 users**.

The first explanation mentioned a 100-user pool, but the final setting was 110. This gives .2.10–119.

Edge:
```text
ip route 192.168.2.0 255.255.255.128 10.0.0.1
ip access-list standard NAT-INTERNAL
 permit 192.168.2.0 0.0.0.127
exit
```

The new network is outside the earlier .0.x and .1.x routes. Both the return route and PAT source list need the new subnet.

## AP and laptop setup

1. Add AccessPoint-PT and connect its Port 0 to the listed switch port.
2. In **Config**, turn wired Port 0 and wireless Port 1 **On**.
3. Set the SSID, channel, WPA2-PSK, AES, and a lab password.
4. Use a different password for Staff and Guest. Do not put the passwords in the docs.
5. On Laptop-PT, turn power Off, replace the wired module with **WPC300N**, then turn power On.
6. Open **Desktop → PC Wireless**, refresh, choose the SSID, and enter the password.
7. Open **IP Configuration → Wireless0 → DHCP**.

This uses a shared wireless password. It does not include 802.1X, RADIUS, roaming, or a wireless capacity survey.

## Checks and results

Guest Wireless0 received **192.168.1.196/26**, gateway .1.193. Gateway Ping passed 2/2. Internal servers .1.2 and .1.3 were blocked, 0/2 each.

Staff Wireless0 received **192.168.2.11**. Gateway .2.1, internal server .1.2, and external server 203.0.113.10 each replied 2/2. The supplied ipconfig ended after the wireless IP, so the full wireless mask/gateway/DNS fields were not collected.

Staff internal-name HTTP and denied Core SSH were reported complete. The edge PAT table later showed the Staff laptop's actual translated HTTP connection.

The /25 has room for the address target. It does not prove one AP can serve all 70 employees and future growth under real load.

[View the evidence](validation.md) · [Review access rules](security.md)

---

[Back to main README](../readme.md) · [Rebuild guide](guide.md) · [Test results](validation.md)
