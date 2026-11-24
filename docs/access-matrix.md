# Access Control Matrix (Before / After ACL)

"Before" = after VLANs, routing, DHCP, DNS and NAT, but before ACLs were applied.
"After" = with `STAFF-IN`, `GUEST-IN` and the VTY `SSH-MGMT-ONLY` access-class applied.
All screenshots are in [`../screenshots/`](../screenshots/).

| Source | Destination | Service | Before ACL | After ACL | Evidence |
|---|---|---|---|---|---|
| Staff | SRV1 (10.10.30.10) | DNS, HTTP | Allowed | Allowed | [09](../screenshots/09-staff-web-dns.png), [15 browser](../screenshots/15-after-staff-to-server-blocked_browser.png) |
| Staff | SRV1 | ICMP | Allowed | Blocked | [14](../screenshots/14-before-staff-to-server.png), [15 ping](../screenshots/15-after-staff-to-server-blocked_ping.png) |
| Staff | Internet | Any | Allowed | Allowed | [10](../screenshots/10-nat-translations.png), [16](../screenshots/16-after-staff-to-internet.png) |
| Staff | Guest, Mgmt | Any | Not tested | Blocked | CLI output, `STAFF-IN` lines 60 and 70 counters ([23](../screenshots/23-r1-access-lists.png)) |
| Guest | Staff | Any | Allowed | Blocked | [11](../screenshots/11-before-guest-to-staff.png), [17](../screenshots/17-after-guest-to-staff-blocked.png) |
| Guest | SRV1 | Any | Allowed | Blocked | [12](../screenshots/12-before-guest-to-server.png), [18](../screenshots/18-after-guest-to-server-admin-blocked.png) |
| Guest | Mgmt | Any | Allowed | Blocked | [13](../screenshots/13-before-guest-to-admin.png), [18](../screenshots/18-after-guest-to-server-admin-blocked.png) |
| Guest | Internet | Any | Allowed | Allowed | [10](../screenshots/10-nat-translations.png), [19](../screenshots/19-after-guest-internet-ok.png) |
| Staff | R1 (SSH) | TCP/22 | n/a | Refused (VTY access-class) | [22](../screenshots/22-ssh-staff-denied.png) |
| Guest | R1 (SSH) | TCP/22 | n/a | Timed out (blocked by `GUEST-IN`) | CLI output only |
| Admin | R1, SW1, SW2 (SSH) | TCP/22 | n/a | Allowed | [21](../screenshots/21-ssh-admin-success.png) |
| Admin | Staff | ICMP | Not tested | Timed out (consistent with stateless ACL) | CLI output only |

## Hardening evidence

| Control | Evidence |
|---|---|
| VLANs and trunks (native VLAN 999, allowed 10,20,30,99) | [02](../screenshots/02-sw1-vlan-brief.png), [03](../screenshots/03-sw2-vlan-brief.png), [04](../screenshots/04-sw1-trunk.png) |
| DHCP renew still works with ACLs applied | [20 staff](../screenshots/20-dhcp-renew-after-acl_staff.png), [20 guest](../screenshots/20-dhcp-renew-after-acl_guest.png) |
| ACL hit counters | [23](../screenshots/23-r1-access-lists.png) |
| Port security (sticky, max 1, violation shutdown) | [24](../screenshots/24-sw2-port-security.png) |
| Port security violation (`Secure-shutdown`, count 1) | [25](../screenshots/25-port-violation-errdisabled.png) |
| Unused ports shut down in VLAN 999 | [26a](../screenshots/26a-sw2-unused-ports.png), [26b](../screenshots/26b-sw1-unused-ports.png) |
| DTP disabled on trunks | [26c](../screenshots/26c-sw1-nonegotiate.png) |

Note: Packet Tracer reports blocked traffic as "Destination host unreachable" from the gateway; real IOS typically reports "administratively prohibited".
The Admin-to-Staff timeout is observed behavior consistent with stateless filtering (Staff replies to the MGMT subnet match `STAFF-IN` line 70). It was not isolated further.