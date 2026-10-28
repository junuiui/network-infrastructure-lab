# Screenshot Index

[Back to main README](../readme.md) · [Architecture](../docs/architecture.md) · [Troubleshooting](../docs/troubleshooting.md)

These images show different stages of the lab. A file named “fixed” can still be an intermediate step. Use [the current address table](../docs/addressing.md) for settings.

## Final layout

![Final topology](final.png)

**final.png** shows four departments, Server Farm, Guest, Staff Wi-Fi, the edge router, external server, and voice router. It shows the layout, not every saved setting or test result.

Used in [README](../readme.md), [Architecture](../docs/architecture.md), [Wireless](../docs/wireless.md), [Guide](../docs/guide.md), and [Validation](../docs/validation.md).

## Early HR setup

![HR phone and PC wiring](1-HR-structure.png)

**1-HR-structure.png** shows two PCs connected through two IP phones, plus a printer and HR switch. Used to explain shared wiring with separate Data/Voice VLANs.

![HR VLAN entries](1-HR-VLAN.png)

**1-HR-VLAN.png** shows VLAN 10 HR-Data and VLAN 110 HR-Voice. VLAN existence alone does not prove routing or phone registration.

## Core and router build

![Early core links](2-main-switch-connect.png)

**2-main-switch-connect.png** shows the early three-department core layout. Some link indicators reflect a build in progress.

![Early router connection](3-router.png)

**3-router.png** adds the 2911 router to that early layout. It predates the later server and wireless sections.

## DHCP: early configuration

![Early DHCP values](dhcp-server-configurations.png)

**dhcp-server-configurations.png** shows old pools, a default pool starting at .1.0 with 512 users, and HR pools that needed correction. It is a troubleshooting reference, not a setup template.

## DHCP: before expansion

![DHCP before subnet expansion](dhcp-server-pool-problem.png)

**dhcp-server-pool-problem.png** shows serverPool limited to .1.6/29 and the old department/Guest address plan.

## DHCP: partial fix

![DHCP during the migration](dhcp-server-pool-fixed.png)

**dhcp-server-pool-fixed.png** shows S/M and IT on new ranges and serverPool /28. HR and Guest still use old ranges. DNS/TFTP values are also from before the later service setup.

Used in [Addressing](../docs/addressing.md), [Troubleshooting](../docs/troubleshooting.md), and [Guide](../docs/guide.md).

---

[Back to main README](../readme.md)
