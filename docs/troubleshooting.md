# Troubleshooting Log

[Back to main README](../readme.md) · [Rebuild guide](guide.md) · [Test results](validation.md)

## How this log is written

Each entry separates the symptom, checks, cause, fix, and result. A capacity risk found in a plan is not the same as a captured outage. A likely cause is not marked as proven without evidence.

## 1. Early DHCP failure

### Symptom

Clients could not get DHCP addresses. They showed APIPA 169.254.x.x or kept waiting.

### Checks and fixes

| Problem | What was found | Action |
| :--- | :--- | :--- |
| Wrong device context | Layer 3 commands were entered on an access switch | Run routing and relay commands on the 3560 core |
| Invalid gateway address | VLAN 99 used .0.192 and VLAN 10 used .0.224, both network addresses | Use .0.193 and .0.225 in the original plan |
| Server port in wrong VLAN | The old Core Gi0/2 server connection was in VLAN 10 | Move it to Server VLAN 50 |
| Missing SVI IPs | VLAN 50, 40, and 140 were unassigned | Add gateway IPs and enable the interfaces |

After the gateways were corrected, the original log reported successful relay to 192.168.1.2 and client leases. These are early addresses. The server later moved to Server-Switch Fa0/1, and the address plan changed.

![Early DHCP settings](../resources/dhcp-server-configurations.png)

*The early screen also shows an oversized default pool and HR ranges that include broadcast addresses.*

## 2. Address capacity and pool limits

### Problem

Server Farm and IT Data had only five device addresses after the gateway, below their six-device targets. Several other subnets did not meet the new 50% growth rule.

### First approach

Expand the DHCP mask and pool, then update the core SVI.

### What else was needed

Changing only the pool would not fix the whole path. The work also required non-overlapping subnets, static-IP changes, server masks, Guest ACL updates, and client lease renewal.

### Result

Representative clients received the new addresses and passed the tests in [Validation](validation.md). There was no full-pool exhaustion or load test. Capacity was checked by address calculation.

HR Voice already met 50% growth with /28. Its later /27 expansion was a choice for more room, not a fix for a proven shortage.

## 3. Core rejected an overlapping subnet

Actual error:
```text
Switch(config-if)#no ip address
Switch(config-if)#ip address 192.168.0.65 255.255.255.192
% 192.168.0.64 overlaps with Vlan120
```

### Cause

192.168.0.65 belongs to ENG Voice 192.168.0.64/26. The DHCP start addresses .2 and .66 are not the subnet boundaries. ENG occupies .0–63 and .64–127.

A second idea, Data .0.129 and Voice .0.162, also overlaps under /26: both are in .0.128–191. The next /26, .0.192–255, was occupied by Guest and HR.

### Fix

Move S/M Data to **192.168.1.64/26** and Voice to **192.168.1.128/26**. Set the two SVIs explicitly.

### Result and lesson

S/M PC received .1.67/26, gateway .1.65, and reached the gateway and servers.

The `no ip address` command can succeed even when the next address command fails. That can leave the SVI without a gateway. Compare the whole network-to-broadcast range, not just the last octet.

## 4. Old DHCP lease after migration

Guest first showed .0.195/27 with gateway .0.193. A later output showed .1.195/26 with gateway .1.193.

Renew the client after saving the pool and SVI changes: select Static, then DHCP again. The old output alone does not prove the new pool is wrong.

Wired .1.195 and wireless .1.196 later received the new Guest settings.

![Guest and HR still on old addresses during the migration](../resources/dhcp-server-pool-fixed.png)

*This “fixed” image is an intermediate step. S/M and IT are moved, while HR and Guest still show old values.*

## 5. Guest gets DHCP but cannot Ping the DHCP server

### Symptom

Guest gateway Ping passed. Pings to .1.2 and .1.3 returned Destination host unreachable from the gateway.

### Check

The seven blocked wired requests matched the Guest ACL's seven hits on the private-network deny rule.

### Result

This is the intended policy. DHCP UDP 68→67 is allowed, but normal access to internal servers is blocked.

The later wireless test showed the same behavior. A fresh ACL counter after the wireless test was not collected.

A match on the final permit rule does not prove Internet or PAT success.

## 6. ENG printer was inside the DHCP range

The initial printer address .0.4 fell inside DHCP .0.2–61. A duplicate lease was possible; an actual duplicate lease was not captured.

The fix was to reduce ENG-Data from 60 to **58 users**, giving .2–59, and move the printer to **.62/26** with gateway .1.

Printer Ping passed 3/3 and DHCP server Ping passed 2/2. The final saved pool limit still needs a screenshot. Ping alone does not confirm that limit.

## 7. First Ping timeout and test mistakes

S/M→Web .1.3 had one timeout followed by two replies. Initial ARP is a possible cause, not a confirmed diagnosis. A later uninterrupted four-reply test was not supplied.

Guest also pinged .1.192, which is the network address. The reply came from .193, so it was not evidence of a host at .192.

`ing` was a typing error for `ping`. `show running config` was corrected to `show running-config`. Neither was a network fault.

## 8. SSH log showed 1.99 after selecting v2

The activation message mentioned SSH 1.99 after `ip ssh version 2`. Instead of judging by that message, the check used `show ip ssh`.

Actual result:
```text
SSH Enabled - version 2.0
Authentication timeout: 120 secs; Authentication retries: 3
```

This confirmed the core SSH version. Login and access rules still needed separate tests.

## 9. Core SSH authentication failed

The connection reached repeated Password prompts, then closed. The local account secret was reset, and VTY `login local` was applied.

The exact original password/input mismatch was not confirmed. The later ACL problem was a separate failure and was not used to explain the earlier authentication prompts.

## 10. SSH refused because the wrong subnet was allowed

Actual ACL:
```text
Standard IP access list SSH-IT-ONLY
 10 permit 192.168.1.0 0.0.0.15
 20 deny any (10 match(es))
```

### Cause

The ACL allowed .1.0–15. Wired IT clients are in .1.16/28, normally .18–27.

### Fix

```text
configure terminal
ip access-list standard SSH-IT-ONLY
 no permit 192.168.1.0 0.0.0.15
 no deny any
 permit 192.168.1.16 0.0.0.15
 deny any
exit
end
```

The permit must come before deny any. SSH then reached Core-Switch#. The final collected ACL showed permit 2 and deny 5.

Connection refused alone is not a diagnosis. The checks also covered SSH status, VTY settings, active users, and ACL counters.

## 11. IT switch authentication failed

SSH .1.28 reached Password prompts but closed. The account secret was reset on **IT-Switch**, and `login local` was applied to VTY 0–15.

The next attempt reached IT-Switch#. Its ACL later showed permit 16 and deny 16.

Each device has its own local account database. Creating an account on Core does not create it on IT-Switch.

## 12. Why HR needs another VLAN 10 IP

Core .0.193 is the HR gateway. HR-Switch .0.220 is the switch's own management IP. They are separate devices in the same subnet.

SSH to .220 succeeded, and its ACL showed permit 2 and deny 15.

## 13. IP HTTP worked but the hostname did not

### Symptom

Ping to the DNS server worked. The PC DNS address was correct. IP-based HTTP worked, but the name returned Hostname unresolved.

### Cause and fix

DNS Service was **Off**. Turning it On fixed the name test.

Check reachability, service state, and the actual record separately. Ping does not prove the DNS service is running.

## 14. Edge router did not accept telephony-service

The current 2911 simulation rejected the command. A separate 2811 accepted `telephony-service ?` with `<cr>`.

Edge routing/PAT stayed in place. The 2811 was used for CME. This is a simulator model/IOS finding, not a claim about every real 2911.

## 15. Phone stayed at Configuring CM List

### Findings

- Voice-Router Fa0/0 used **192.168.1.3**, duplicating Web-Server.
- CME had no `ip source-address`.
- `show ephone` was empty.

### Fix

Set Fa0/0 to **192.168.1.4/28**, add `ip source-address 192.168.1.4 port 2000`, run `create cnf-files`, check Voice DHCP TFTP .4, and restart the phone.

### Result

HR .0.226 registered as extension 1001. Eight phones and calls between HR/ENG and S/M/IT were later reported complete.

## 16. Syslog severity commands were rejected

Both `logging trap informational` and `logging trap 6` failed. Help still listed trap. `logging trap debugging` worked and appeared in `show logging`.

The server received three core messages. The earlier numeric alternative was not valid in this environment. The rebuild guide uses the working command.

## Current state

Staff Wi-Fi, Guest wireless checks, actual PAT entries, and Syslog reception are complete at the evidence levels shown in [Validation](validation.md). The final .pkt exists in the repository.

Some saved settings, backup files, and restore results remain unchecked. They are not marked as failures or invented as completed tests.

---

[Back to main README](../readme.md) · [Rebuild guide](guide.md) · [Test results](validation.md)
