# Company Network Infrastructure Lab

**Cisco Packet Tracer | Networking, troubleshooting, and recovery**

I built a simulated company network to practice the network tasks behind day-to-day IT support. The company plan has 70 employees across HR, Engineering, Sales/Marketing, and IT. I used a few devices in each department to test the setup.

![Company network with department switches, servers, IP phones, and Staff and Guest Wi-Fi](resources/final.png)

*Final layout. The external server represents a network outside the company, not a live Internet connection.*

## What I built

| Area | What I did | Why |
| :--- | :--- | :--- |
| Network separation | Created department Data and Voice VLANs, plus Server, Staff, and Guest networks | Keep different types of traffic separate |
| IP addressing | Reworked the subnet plan, DHCP pools, and static addresses | Allow growth and avoid address conflicts |
| Network services | Set up central DHCP, relay, DNS, and web services | Give clients the right settings and access by name |
| External access | Added static routes and PAT on the edge router | Practice traffic flow to an external test network |
| Access control | Blocked Guest access to internal networks and limited SSH to wired IT clients | Separate visitor traffic and protect device management |
| Wireless and phones | Added Staff/Guest Wi-Fi and eight IP phones across four departments | Test common office connections |
| Logs and recovery | Collected core Syslog events and tested a TFTP configuration restore | Practice checking events and recovering a saved setting |

## Connecting users and services

The core switch handles routing between VLANs and relays DHCP requests to one server. Staff use internal DNS to open the company test website. Guests use an external test DNS server and cannot reach internal servers.

Staff and Guest Wi-Fi use separate APs, SSIDs, and VLANs. A Staff laptop and a wired IT PC were both shown sharing the edge router's external address through PAT.

## Managing switches and phones

I configured SSH on the Core, IT, and HR switches and tested allowed and blocked connections. The other access switches were left out of this repeated SSH setup.

A separate 2811 router runs Call Manager Express for two phones in each department. I tested all eight registrations and calls between departments.

![HR PCs connected through two IP phones to the department switch](resources/1-HR-structure.png)

*PCs connect through the phones. Data and Voice traffic use separate VLANs.*

## Troubleshooting and recovery

The main problems were overlapping subnets, DHCP and static-IP overlap risk, a wrong SSH permit network, DNS Service being Off, and a duplicate voice-router address. I checked the symptoms, narrowed down the cause, changed the settings, and tested again.

I also backed up the IT switch to TFTP, changed its hostname, and loaded the saved file into running-config. The upload and download both returned **1,979 bytes**. The hostname returned to IT-Switch, the management interface stayed up, and SSH login worked.

## Results and scope

- Checked client addressing, routed connections, Guest blocking, and SSH access.
- Confirmed PAT entries for wired and wireless clients.
- Tested internal and external DNS/web access and calls between departments.
- Confirmed core messages arrived at the Syslog server.
- Checked a manual IT-Switch backup and running-config restore.

This is a Packet Tracer lab, not production experience. Syslog covers event collection, not a full monitoring dashboard. The restore test merged a saved configuration into a running switch; it was not an erased-device or reboot recovery test.

**Lab files:** [Final network](fake-company-final.pkt) · [Restore test](fake-company-restore-test.pkt)

## Detailed documentation

- **[HowTo.md](HowTo.md)** — Devices, port connections, settings, commands, and step-by-step checks.
- **[ProblemSolving.md](ProblemSolving.md)** — Problems, investigation steps, fixes, and actual test results.
