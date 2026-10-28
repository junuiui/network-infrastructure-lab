# Routing, DHCP, PAT, and DNS

[Back to main README](../readme.md) · [Rebuild guide](guide.md) · [Test results](validation.md)

## What each device does

| Device | Job |
| :--- | :--- |
| Core switch | VLAN gateways, routing between VLANs, and DHCP relay |
| DHCP-Server, 192.168.1.2 | IP leases and internal DNS |
| Edge router | External routing and PAT |
| Internal Web, 192.168.1.3 | Company test website |
| External-Web, 203.0.113.10 | External test website and Guest DNS |

## DHCP relay

A client sends a DHCP broadcast before it has an IP. Routers do not normally forward that broadcast. Each client VLAN SVI uses:

```text
ip helper-address 192.168.1.2
```

The core forwards the request to the central server. The server uses the relay information to select the right pool. See [the pool table](addressing.md).

VLAN 50 and the DHCP server are in the same subnet, so VLAN 50 does not need a helper. The old config kept one there; the rebuild guide leaves it out.

## Core-to-edge link

| Interface | Address |
| :--- | :--- |
| Core Gi0/1 | 10.0.0.1/30 |
| Edge Gi0/0 | 10.0.0.2/30 |
| Edge Gi0/1 | 203.0.113.1/24 |
| External-Web | 203.0.113.10/24, gateway 203.0.113.1 |

The /30 gives two usable addresses for the two ends. Core Gi0/1 was changed from a trunk to a routed port.

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

- `no switchport` turns the core port into a Layer 3 port.
- The core default route sends unknown destinations to the edge.
- The edge static routes send return traffic to the core.
- The /24 return routes summarize internal networks. They do not change client subnet masks.
- The Staff /25 route was added when Staff Wi-Fi was built.
- External-Web is directly connected, so the edge does not need a default route to reach it.

The collected edge route table showed Connected/Local entries for the transit link and Static routes for 192.168.0.0/24 and 192.168.1.0/24. Staff return traffic was later tested through the final PAT connection.

![Early router connection](../resources/3-router.png)

*Early build image. The final network also includes servers and two wireless networks.*

## PAT

PAT lets several internal clients share one outside IP. The router tracks port information so replies go back to the right client.

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

`NAT-INTERNAL` selects sources for translation. It is not applied as an interface blocking rule. `inside` and `outside` mark the two sides. `overload` enables address sharing.

After opening the external website:
```text
show ip nat translations
show ip nat statistics
```

Actual output:
```text
Pro Inside global      Inside local        Outside local      Outside global
tcp 203.0.113.1:1027   192.168.2.11:1027   203.0.113.10:80    203.0.113.10:80
tcp 203.0.113.1:1031   192.168.1.19:1031   203.0.113.10:80    203.0.113.10:80
```

- **Inside local:** Original client address.
- **Inside global:** Address seen outside, 203.0.113.1.
- **Outside global:** External server address.
- **:80:** HTTP port.

Both clients share 203.0.113.1. The statistics showed two dynamic extended translations, 20 hits, eight misses, and six expired translations. A miss is not a count of lost packets or failed user connections.

A successful web request alone would not prove PAT in this lab. Routing could also carry traffic without translation. The translation table is the direct evidence.

## Internal and Guest DNS

DNS maps a name to an IP. An A record maps to IPv4. DHCP tells each client which DNS server to ask.

| Clients | DNS server | Record |
| :--- | :--- | :--- |
| HR/ENG/S/M/IT Data and Staff Wi-Fi | 192.168.1.2 | intranet.fakecompany.test → 192.168.1.3 |
| Guest | 203.0.113.10 | www.external.test → 203.0.113.10 |

On each DNS server:

1. Open **Services → DNS** and turn the service **On**.
2. Add the name without `http://`, choose **A Record**, enter the IP, and click **Add**.
3. Check that the record appears in the list.
4. Set the DNS field in the matching DHCP pool and click **Save**.
5. Renew the client lease and check `ipconfig /all`.
6. Test the name with `nslookup` and the browser.

Internal test: `http://intranet.fakecompany.test`. Guest test: `http://www.external.test`.

The first internal name test failed even though Ping and IP-based HTTP worked. DNS Service was Off. Turning it On fixed the name test. Staff Data pool tests and Guest external DNS tests were then reported complete.

Guest still cannot reach internal DNS. It uses the external test server instead. These records do not provide general public DNS recursion.

## Verification limits

The PAT table is collected evidence. Some DNS, HTTP, and route checks are completion reports without raw output. See [Validation](validation.md).

The external server is a simulated destination, not real Internet access.

---

[Back to main README](../readme.md) · [Rebuild guide](guide.md) · [Test results](validation.md)
