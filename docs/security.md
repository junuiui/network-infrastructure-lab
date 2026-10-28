# Guest ACL and SSH Management

[Back to main README](../readme.md) · [Rebuild guide](guide.md) · [Test results](validation.md)

## Guest policy

The core applies GUEST-IN inbound on VLAN 99. It allows DHCP and Ping to the Guest gateway, blocks private destination networks, then allows other destinations.

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

The order matters. The first matching rule is used. DHCP must be allowed before the internal-network deny rules.

| Test | Expected |
| :--- | :--- |
| Receive DHCP address | Allowed |
| Ping 192.168.1.193 | Allowed |
| Reach internal servers or departments | Blocked |
| Reach external test server | Allowed when routing/PAT are ready |

This does not block traffic between clients in the same Guest VLAN. It is not a stateful firewall or AP client isolation. NAT also does not replace a firewall policy.

## SSH scope

| Device | SSH address | Status |
| :--- | :--- | :--- |
| Core | 192.168.1.17 | Login and ACL checks passed |
| IT switch | 192.168.1.28 | Login and ACL checks passed |
| HR switch | 192.168.0.220 | Login and ACL checks passed |
| ENG / S/M / Server / Guest switches | Not assigned for this task | Repeated setup left out |
| Edge / Voice routers | Not configured for SSH in this task | Not tested |

The purpose was to test SSH on the core and sample access switches. It was not a full management rollout.

## Management IPs

IT switch:
```text
configure terminal
interface vlan 40
 ip address 192.168.1.28 255.255.255.240
 no shutdown
exit
ip default-gateway 192.168.1.17
end
```

HR switch:
```text
configure terminal
interface vlan 10
 ip address 192.168.0.220 255.255.255.224
 no shutdown
exit
ip default-gateway 192.168.0.193
end
```

The management SVI is the switch's own address. It does not replace the client gateway.

For example, Core VLAN 10 uses .0.193 as the HR gateway. HR-Switch VLAN 10 uses .0.220 so an admin can connect to that switch. A Layer 2 switch can forward frames without a management IP, but needs an IP for SSH.

## Local account, keys, and access policy

On each selected device, after setting its hostname:
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

Replace `<LAB_PASSWORD>` with a lab-only password. At the RSA prompt, 2048 was the requested key size; the actual key-size output was not collected.

For IT and HR:
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

For Core:
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

Check the full core config for any extra VTY range. If one exists, apply the same policy to it.

| Setting | Purpose |
| :--- | :--- |
| username / secret | Create a local account and store a password hash |
| privilege 15 | Give full admin access |
| ip domain-name | Used with the hostname for RSA setup |
| ip ssh version 2 | Select SSH v2 |
| line vty 0 15 | Select 16 remote terminal lines |
| login local | Use this device's local account database |
| transport input ssh | Allow SSH on these lines |
| access-class ... in | Limit the source networks for remote sessions |
| exec-timeout 5 0 | End idle sessions after five minutes |

VTY numbers are not switch ports or VLAN IDs. They select existing remote terminal lines. Privilege 15 is an access level; the 15 in VTY 0–15 is a line number.

## Who can manage the devices?

SSH-IT-ONLY allows **192.168.1.16/28**, the wired IT network. It is applied to VTY lines, so it does not block normal routing between departments.

Staff Wi-Fi uses 192.168.2.0/25 and is outside the permit range. It can reach internal web services but cannot manage switches over SSH. Guest traffic is also blocked by the Guest ACL.

## Actual checks

- Core login succeeded after the source subnet was corrected.
- IT login succeeded after its local account was reset.
- HR login succeeded.
- ACL counters showed both permitted and denied traffic.
- Staff Wi-Fi SSH blocking was reported complete.
- Passwords and hashes are not included here.

The supplied PC login text did not include source IPs. Test sources follow the IT/HR/S/M steps used during the lab. See [Troubleshooting](troubleshooting.md) and [Validation](validation.md) for the exact evidence.

---

[Back to main README](../readme.md) · [Rebuild guide](guide.md) · [Test results](validation.md)
