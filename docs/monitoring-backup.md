# Syslog and Configuration Backups

[Back to main README](../readme.md) · [Rebuild guide](guide.md) · [Test results](validation.md)

## Monitoring scope

This lab collects **core device events on a central Syslog server**. It does not include a performance dashboard, SNMP polling, scheduled availability checks, or automatic alerts.

Syslog is useful for checking when a port changed state or a configuration event happened. It does not automatically capture every DHCP lease, failed login, blocked packet, or website visit.

## Syslog setup

On DHCP-Server **192.168.1.2**, open **Services → SYSLOG → On**.

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

| Command | Purpose |
| :--- | :--- |
| service timestamps log datetime msec | Add the device time to messages |
| logging 192.168.1.2 | Set the receiving server |
| logging trap debugging | Send levels 0–7 |
| logging on | Enable logging |
| show logging | Check the setup and sending status |

The word `debugging` sets the message threshold. It does not turn on separate `debug` diagnostics.

### Command errors

`logging trap informational` and `logging trap 6` were rejected. The help listed `trap`, and `logging trap debugging` worked. The record does not treat the rejected forms as applied settings.

### Reception test

Enter and leave configuration mode:
```text
configure terminal
end
```

Then check the server SYSLOG list for a new CONFIG_I event.

Collected core output:
```text
Syslog logging: enabled
Trap logging: level debugging
Logging to 192.168.1.2 (udp port 514 ... link up)
2 message lines logged
```

The server screenshot showed three messages from **192.168.1.1**, the core's Server VLAN address:

- Two `SYS-5-CONFIG_I` events.
- One `SYS-6-LOGGINGHOST_STARTSTOP` event.

This confirms the server received messages, not just that the core had a destination set. The screenshot was supplied in the chat and is not currently a file in resources.

CONFIG_I records a configuration event. It is not a full audit trail of each command or before/after values. The device and server dates were not synchronized with the real date. An unused-port shutdown test was suggested but was not the collected test.

## Save versus backup

| Action | Location | Why |
| :--- | :--- | :--- |
| copy running-config startup-config | Inside the device | Keep settings after a restart |
| copy running-config tftp: | TFTP server | Keep a separate config file |
| Save the .pkt | Packet Tracer project | Keep devices, services, and the lab state |

On DHCP-Server, open **Services → TFTP → On**. This backup service is separate from phone TFTP on Voice-Router .4.

On the device:
```text
enable
copy running-config startup-config
copy running-config tftp:
```

Enter **192.168.1.2** as the remote host, then a device-specific file name.

| Device | File name | Evidence |
| :--- | :--- | :--- |
| Core | Core-Switch-final.cfg | Backup task confirmed complete by the lab owner; individual capture not supplied |
| Edge | Edge-Router-final.cfg | Backup task confirmed complete by the lab owner; individual capture not supplied |
| Voice | Voice-Router-final.cfg | Backup task confirmed complete by the lab owner; individual capture not supplied |
| IT | IT-Switch-final.cfg | Startup [OK], 1,979-byte upload and download checked; restore tested |

HR and Server switch startup saves were also requested. Their individual outputs were not supplied.

The lab owner later explicitly confirmed the backup task was complete. Individual Core/Edge/Voice transfer results and a server file-list screenshot were not supplied. The IT-Switch file is directly confirmed by successful upload and retrieval.

Check both the transfer result and the server TFTP file list. A successful startup save does not prove a successful TFTP transfer.


## IT-Switch backup and restore test

The supplied CLI output confirms a successful TFTP backup and a restore into the running configuration.

| Check | Actual result | Status |
| :--- | :--- | :--- |
| Save startup-config | Building configuration... [OK] | Output checked |
| Upload IT-Switch-final.cfg to 192.168.1.2 | [OK - 1979 bytes] | Output checked |
| Change hostname | IT-Switch → RESTORE-TEST | Output checked |
| Read the backup into running-config | [OK - 1979 bytes] | Output checked |
| Restore hostname | Prompt returned to IT-Switch# | Output checked |
| Management SVI | VLAN 40, 192.168.1.28, up/up | Output checked |
| SSH ACL | Permit 192.168.1.16 0.0.0.15, then deny any | Output checked |
| Client Ping to 192.168.1.28 | 2/3 replies; first request timed out | Replies checked |
| Client SSH to 192.168.1.28 | Login reached IT-Switch# | Output checked |

Backup output:

```text
IT-Switch#copy running-config tftp:
Address or name of remote host []? 192.168.1.2
Destination filename [IT-Switch-confg]? IT-Switch-final.cfg
Writing running-config...!!
[OK - 1979 bytes]
1979 bytes copied in 0 secs
```

Restore output:

```text
IT-Switch(config)#hostname RESTORE-TEST
RESTORE-TEST(config)#end
RESTORE-TEST#copy tftp: running-config
Address or name of remote host []? 192.168.1.2
Source filename []? IT-Switch-final.cfg
Destination filename [running-config]?

Accessing tftp://192.168.1.2/IT-Switch-final.cfg...
Loading IT-Switch-final.cfg from 192.168.1.2: !
[OK - 1979 bytes]
1979 bytes copied in 0 secs
IT-Switch#
```

The restored VLAN 40 interface was 192.168.1.28 and up/up. The ACL output still showed permit and deny entries, with 16 matches each. Those counters confirm the entries were present; they do not prove a new denied login was tested after the restore.

This test used **copy tftp: running-config**, which merges the file into the active configuration. The hostname changed back, proving that the downloaded file applied a saved value. The IP and ACL checks show the expected configuration was present afterward; those settings were not deliberately removed before the test.

This was not an erased-device rebuild, a full configuration replacement, or a reboot recovery test. The first Ping timeout is recorded as observed; its cause was not confirmed. Successful retrieval also proves this backup file was present and readable on the server, even without a file-list screenshot.

## Final file

[fake-company-final.pkt](../fake-company-final.pkt) is in the repository. It was not reopened by the documentation assistant to check every saved setting. The later IT-Switch restore was run by the lab owner, and its CLI results were supplied.

The TFTP server is inside the same .pkt. This demonstrates a backup to another lab device, not an independent backup that survives loss of the host file. TFTP also does not encrypt transfers. Real backups need a protected path and storage location.

## Next learning area

Availability checks, resource graphs, service alerts, and incident handling can be practiced in a separate VM monitoring lab. They are not claimed as completed here.

[Check the final evidence](validation.md)

---

[Back to main README](../readme.md) · [Rebuild guide](guide.md) · [Test results](validation.md)
