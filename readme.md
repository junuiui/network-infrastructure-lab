# Enterprise Network Infrastructure Lab

A Cisco Packet Tracer enterprise network lab covering VLAN segmentation, Layer 3 switching, centralized DHCP, access control, SSH, simulated external connectivity, PAT, DNS, and IP telephony.

**Updated: 2026-10-27.** Detailed configuration history, old/new addressing tables, troubleshooting and evidence are in [1-Fake-company.md](1-Fake-company.md).

## Architecture

- Core: Cisco 3560 multilayer switch for inter-VLAN routing and DHCP relay.
- Access: departmental, server and guest switches.
- Edge: Cisco 2911 for static routing and PAT.
- Voice: separate Cisco 2811 for CME, phone configuration and SCCP registration.
- Server-PT: DHCP/internal DNS, internal Web, and simulated external HTTP/DNS.
- Guest AP with wireless client.
- Server-PT is not Windows Server or Active Directory. This project is separate from the VMware Windows Server AD lab.

## Current Address Plan

Original addresses are preserved in the detailed guide for comparison.

| Network | Data VLAN / Subnet | Voice VLAN / Subnet |
| :--- | :--- | :--- |
| HR | 10 / 192.168.0.192/27 | 110 / 192.168.0.224/27 |
| Engineering | 20 / 192.168.0.0/26 | 120 / 192.168.0.64/26 |
| Sales/Marketing | 30 / 192.168.1.64/26 | 130 / 192.168.1.128/26 |
| IT | 40 / 192.168.1.16/28 | 140 / 192.168.1.32/28 |
| Server Farm | 50 / 192.168.1.0/28 | — |
| Guests | 99 / 192.168.1.192/26 | — |

Core–Edge transit: 10.0.0.0/30. Simulated external network: 203.0.113.0/24.

## Implemented Features and Evidence

| Feature | Status |
| :--- | :--- |
| VLANs, trunks, inter-VLAN routing, centralized DHCP relay | Representative configuration and client outputs collected |
| Address-capacity redesign and static/DHCP separation | Detailed migration and tests documented; final ENG DHCP limit requires confirmation |
| Guest wired/wireless isolation | DHCP, Gateway reachability, internal Ping denial and ACL evidence collected |
| SSH | Core, IT and HR login/ACL evidence collected; remaining switches intentionally omitted as repetitive practice |
| Core–Edge routing | Router route output collected; IT reachability reported complete |
| PAT and simulated external HTTP | User-reported complete; translation/statistics evidence pending |
| Internal and Guest DNS | User-reported name-based access complete; DNS Service Off issue resolved |
| VoIP | HR phone REGISTERED output collected; eight phones and interdepartment calls reported complete |

Reported completion is distinguished from supplied raw evidence. The latest .pkt has not been independently rerun during documentation updates.

## Troubleshooting Highlights

- Insufficient subnet capacity and DHCP/static address overlap.
- Overlapping subnet rejected during S/M migration.
- Incorrect SSH source subnet causing connection refusal.
- Local SSH authentication failures resolved after account configuration reset.
- DNS records present while DNS Service was Off.
- Current 2911 simulation lacking telephony-service; separate 2811 selected after capability check.
- Voice-Router IP duplicated Web Server address and CME source-address was missing; corrected before phone registration.

See the detailed guide for actual commands, diagnostic outputs, actions and test results.

## Remaining Work

- Staff Wi-Fi and its VLAN, DHCP, access policy, return route and NAT coverage.
- Monitoring: Syslog first, supported SNMP checks if useful. Neither is implemented yet.
- Manual TFTP configuration backup, deferred until the end.
- Final evidence capture, configuration saves and latest .pkt verification.
- Python automation is excluded from the current project scope.

## Lab Boundaries

The external server represents a simulated Internet destination. Real Google/public Internet access from Packet Tracer's simulated clients is not implemented or supported through this topology; it requires a separate suitable lab environment.

External Netmiko, PRTG or Zabbix integration is not implemented. No automated-backup or performance-improvement claim is made.
