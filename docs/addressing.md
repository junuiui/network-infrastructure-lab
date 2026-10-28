# IP Addressing and DHCP Plan

[Back to main README](../readme.md) · [Rebuild guide](guide.md) · [Test results](validation.md)

## Design rule

Plan for the required devices, add 50% growth, then add the gateway and other network addresses.

```text
Required usable addresses = ceil(device count × 1.5) + gateway and other infrastructure IPs
```

This means 50% more devices than today. It does not mean leaving half the subnet empty.

A /28 has 16 total addresses and 14 usable addresses. With one gateway, 13 remain for devices. HR Voice needed eight phones, so its old /28 already met the growth rule. It was still expanded to /27 by choice.

Server Farm and IT Data had a different problem: their old /29 allowed only five devices after the gateway, below the original six-device target. This was a design-capacity issue. The lab did not show every pool running out of leases.

## Old and new subnets

The usable range includes the gateway. It is not the DHCP range.

| Network | VLAN | Old subnet | Old gateway | New subnet | New gateway | New mask | New usable range |
| :--- | ---: | :--- | :--- | :--- | :--- | :--- | :--- |
| ENG-Data | 20 | 192.168.0.0/26 | 192.168.0.1 | 192.168.0.0/26 | 192.168.0.1 | 255.255.255.192 | 192.168.0.1–62 |
| ENG-Voice | 120 | 192.168.0.64/26 | 192.168.0.65 | 192.168.0.64/26 | 192.168.0.65 | 255.255.255.192 | 192.168.0.65–126 |
| S/M-Data | 30 | 192.168.0.128/27 | 192.168.0.129 | 192.168.1.64/26 | 192.168.1.65 | 255.255.255.192 | 192.168.1.65–126 |
| S/M-Voice | 130 | 192.168.0.160/27 | 192.168.0.161 | 192.168.1.128/26 | 192.168.1.129 | 255.255.255.192 | 192.168.1.129–190 |
| Guests | 99 | 192.168.0.192/27 | 192.168.0.193 | 192.168.1.192/26 | 192.168.1.193 | 255.255.255.192 | 192.168.1.193–254 |
| HR-Data | 10 | 192.168.0.224/28 | 192.168.0.225 | 192.168.0.192/27 | 192.168.0.193 | 255.255.255.224 | 192.168.0.193–222 |
| HR-Voice | 110 | 192.168.0.240/28 | 192.168.0.241 | 192.168.0.224/27 | 192.168.0.225 | 255.255.255.224 | 192.168.0.225–254 |
| Server Farm | 50 | 192.168.1.0/29 | 192.168.1.1 | 192.168.1.0/28 | 192.168.1.1 | 255.255.255.240 | 192.168.1.1–14 |
| IT-Data | 40 | 192.168.1.8/29 | 192.168.1.9 | 192.168.1.16/28 | 192.168.1.17 | 255.255.255.240 | 192.168.1.17–30 |
| IT-Voice | 140 | 192.168.1.16/29 | 192.168.1.17 | 192.168.1.32/28 | 192.168.1.33 | 255.255.255.240 | 192.168.1.33–46 |
| Staff Wi-Fi | 60 | Not built | — | 192.168.2.0/25 | 192.168.2.1 | 255.255.255.128 | 192.168.2.1–126 |

The final subnets do not overlap. After moving S/M, 192.168.0.128/26 is not assigned in the final plan. Old client settings may still need renewal.

## Final DHCP pools

On DHCP-Server, open **Services → DHCP**. Select an existing pool and click **Save** after changing it. Use **Add** for a new pool.

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

ENG-Data **58** is the final target. The last pool screenshot collected before that change showed 60. Printer Ping passed, but a final pool screenshot is still needed to confirm the saved limit.

| Pool group | DNS server | TFTP server |
| :--- | :--- | :--- |
| HR/ENG/S/M/IT Data and Staff-WiFi | 192.168.1.2 | 0.0.0.0 |
| Guests | 203.0.113.10 | 0.0.0.0 |
| Four Voice pools | No voice DNS requirement set in this lab | 192.168.1.4 |

The basic serverPool could not be removed in the earlier workflow. It was limited to one address. The proposed move to .14 was not used; the checked screen shows **.6**. Keep .6 out of static server assignments.

## Static and management IPs

| Device | Old setting | Final setting | Evidence |
| :--- | :--- | :--- | :--- |
| DHCP server | 192.168.1.2/29, GW .1 | 192.168.1.2/28, GW .1 | Pool screen checked; NIC mask output pending |
| Web server | 192.168.1.3/29, GW .1 | 192.168.1.3/28, GW .1 | ipconfig checked |
| Voice router | Added later; .3 was entered by mistake | 192.168.1.4/28, next hop .1 | Phone registration checked after correction |
| IT printer | 192.168.1.14/29, GW .9 | 192.168.1.30/28, GW .17 | Ping checked |
| S/M printer | Not collected | 192.168.1.126/26, GW .1.65 | Setup given; test pending |
| HR printer | Not collected | 192.168.0.221/27, GW .193 | Ping checked |
| Second HR printer reserve | Not collected | 192.168.0.222/27, GW .193 | Not tested |
| ENG printer | Initial record: 192.168.0.4 | 192.168.0.62/26, GW .1 | Ping checked |
| IT switch | No management IP | 192.168.1.28/28, GW .17 | SSH checked |
| HR switch | No management IP | 192.168.0.220/27, GW .193 | SSH checked |

ENG .0.60–62 are outside the final DHCP pool. HR .0.220 is now used by the switch, not a free printer address.

## Change order

1. Remove the old IT Data/Voice SVI addresses and move IT first.
2. Expand Server Farm after IT leaves the old range.
3. Move both S/M networks to 192.168.1.x.
4. Move Guest and update its ACL.
5. Move HR Data into the old Guest area.
6. Expand HR Voice after the old HR Data address is free.
7. Add Staff Wi-Fi and update the edge return route and PAT list.

Changing only the DHCP mask would leave gateway, static IP, ACL, and route settings out of sync.

## DHCP screenshots: build history

![Early DHCP configuration with an oversized default pool](../resources/dhcp-server-configurations.png)

*Early setup. serverPool starts at a network address with 512 users, and HR pools include too many leases. Do not copy these values.*

![DHCP pools before the subnet expansion](../resources/dhcp-server-pool-problem.png)

*Before expansion. The default pool was reduced, but Server/IT capacity and the growth plan still needed work.*

![DHCP pools during the migration](../resources/dhcp-server-pool-fixed.png)

*Partway through the fix. S/M and IT have moved, but HR and Guest still show old values. DNS and phone TFTP are also not set yet. The final tables above are the current reference.*

## Original IP table

This table is kept for comparison. Do not use it for a new build.

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


[Read the problems and fixes](troubleshooting.md) · [Check the evidence](validation.md)

---

[Back to main README](../readme.md) · [Rebuild guide](guide.md) · [Test results](validation.md)
