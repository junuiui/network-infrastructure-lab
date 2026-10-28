# Enterprise Network Infrastructure Lab

**Cisco Packet Tracer | Network setup, testing, and troubleshooting**

I built a small company network with separate networks for HR, Engineering, Sales/Marketing, IT, servers, and guests. The lab covers IP addressing, DHCP, DNS, routing, access control, Wi-Fi, and IP phones.

[**Start the rebuild guide**](docs/guide.md) · [**Open the final Packet Tracer file**](fake-company-final.pkt) · [**View test results**](docs/validation.md) · [**Read the troubleshooting log**](docs/troubleshooting.md)

![Final lab layout with four departments, servers, an edge router, and two wireless networks](resources/final.png)

*Final layout. The external server represents a network outside the company. It is not a live Internet connection.*

## Project overview

| Area | What I built |
| :--- | :--- |
| Network | Layer 3 core switch, department access switches, and an edge router |
| Addressing | Separate Data and Voice VLANs, with room for growth |
| Services | Central DHCP and relay, internal and external DNS, and web servers |
| Security | Guest isolation and SSH access limited to the wired IT network |
| Wireless | Separate Staff and Guest SSIDs and VLANs |
| Phones | A separate 2811 router running CME for eight IP phones |
| Operations | Central Syslog collection and manual TFTP backup practice |

The company plan is based on 70 employees. I used a few devices in each department to test the design instead of adding every planned device.

## 01. Architecture and IP addressing

The core switch routes traffic between VLANs. The edge router handles traffic to the simulated external network. A separate voice router manages phone registration and extensions.

I reviewed the original address plan, added growth capacity, and separated DHCP ranges from static addresses. The old and new IP tables are both kept in the docs.

**[Network layout and port map](docs/architecture.md)** · **[Old and new IP plans](docs/addressing.md)**

## 02. Routing, DHCP, PAT, and DNS

A central server assigns IP addresses through DHCP relay. Staff use internal DNS. Guests use the external test DNS server.

The PAT output shows a wired IT PC and a Staff Wi-Fi laptop sharing the edge router's external address, `203.0.113.1`.

**[Service setup and traffic flow](docs/network-services.md)**

## 03. Access control and SSH

The Guest ACL blocks access to private internal networks. SSH is set up on the core, IT, and HR switches. Only the wired IT subnet can connect.

I left SSH off the other access switches because the setup would repeat the same steps.

**[Guest ACL and SSH setup](docs/security.md)**

## 04. Staff and Guest Wi-Fi

Staff and guests use separate APs, SSIDs, and VLANs. Staff can reach internal services. Guests can reach the external test server but cannot reach internal servers. Both wireless networks are blocked from device management over SSH.

**[Wireless setup and tests](docs/wireless.md)**

## 05. IP phones

A 2811 router runs Call Manager Express. I set up eight extensions for two phones in each department.

The collected output confirms one HR phone registered. Full registration and calls between departments were reported complete during the lab.

**[CME, phone registration, and calls](docs/voip.md)**

## 06. Logs and backups

The Syslog server received events from the core switch. I also saved the IT switch configuration and completed a manual TFTP backup.

The docs separate confirmed results from reported results. Individual Core, Edge, and Voice backup files still need a file-list check.

**[Syslog and configuration backups](docs/monitoring-backup.md)**

## 07. Troubleshooting and results

The main issues included overlapping subnets, DHCP/static IP overlap, an incorrect SSH ACL, DNS being turned off, and a duplicate voice-router IP.

Each record explains the problem, the checks, the fix, and the result.

**[Troubleshooting log](docs/troubleshooting.md)** · **[Test results and remaining checks](docs/validation.md)**

## Build it again

The guide starts from a blank Packet Tracer file. It includes the device list, cables and ports, commands, GUI settings, and checks after each stage.

**[Open guide.md](docs/guide.md)**

## Scope and evidence

- **Output checked:** Client IP and Ping results, Guest ACL counters, SSH logins and counters, routing entries, one phone registration, Staff Wi-Fi Ping tests, PAT entries, and Syslog reception.
- **Reported complete:** Some DNS/web tests, all eight phone registrations and calls, Staff Wi-Fi SSH blocking, and the IT switch TFTP backup.
- **Still to check:** Some final settings, individual backup files, and a restore test. The final `.pkt` file is in this repository, but it was not rerun during the documentation update.

The external network is simulated. Server-PT is not Windows Server or Active Directory. Syslog collects events; this lab does not include a performance dashboard or automatic alerts. Python automation is outside the final scope.

## Documentation

| File | What it covers |
| :--- | :--- |
| [guide.md](docs/guide.md) | Rebuild the final design from the start |
| [architecture.md](docs/architecture.md) | Devices, roles, cables, and ports |
| [addressing.md](docs/addressing.md) | Old/new subnets, DHCP pools, and static IPs |
| [network-services.md](docs/network-services.md) | Routing, DHCP, PAT, and DNS |
| [security.md](docs/security.md) | Guest ACL and SSH |
| [wireless.md](docs/wireless.md) | Staff and Guest Wi-Fi |
| [voip.md](docs/voip.md) | CME, extensions, and phone tests |
| [monitoring-backup.md](docs/monitoring-backup.md) | Syslog and manual backup |
| [troubleshooting.md](docs/troubleshooting.md) | Problems, checks, fixes, and results |
| [validation.md](docs/validation.md) | Evidence and remaining checks |
| [Implementation history](docs/archive/implementation-log.md) | Earlier setup and changes |
| [Screenshot index](resources/readme.md) | What each image shows and when it was taken |

**Lab record date: October 27, 2026.** Each detailed page links back to this README.
