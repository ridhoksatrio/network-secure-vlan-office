# Troubleshooting Log

Real issues encountered while building this lab, with diagnosis and fix.

---

## Entry 1: DHCP skipped the first free address

| Field | Detail |
|---|---|
| Phase | 3 (DHCP, DNS, NAT) |
| Symptom | First DHCP lease was 10.10.10.12 instead of the first free address, 10.10.10.11 |
| Diagnosis | `show ip dhcp binding` showed no lease for .11; `show ip dhcp conflict` listed 10.10.10.11 (detected by ping) |
| Root cause | A PC still had the static IP 10.10.10.11 from the VLAN test phase, so the DHCP server's conflict detection skipped it |
| Fix | Switched the PCs to DHCP, then ran `clear ip dhcp conflict *` |
| Lesson | Conflict detection protects against duplicate IPs, but leftover static IPs can silently shrink the pool |

---

## Entry 2: SSH server never started

| Field | Detail |
|---|---|
| Phase | 4 (ACL + hardening) |
| Symptom | `ssh -l admin 10.10.99.1` returned "Connection closed by foreign host"; `show ip ssh` showed "SSH Disabled" |
| Diagnosis | `crypto key generate rsa modulus 2048` returned "Invalid input" at `modulus`; no RSA keys existed, so `ip ssh version 2` failed too |
| Root cause | Packet Tracer does not accept the `modulus` keyword; key size is asked interactively |
| Fix | Ran `crypto key generate rsa`, answered 2048 at the prompt, then `ip ssh version 2` |
| Lesson | Packet Tracer IOS differs from real IOS; verify with `show ip ssh` before testing access rules |

---

## Entry 3: Placeholder pasted as the real password

| Field | Detail |
|---|---|
| Phase | 4 (ACL + hardening) |
| Symptom | `enable` prompted for a password I never remember setting |
| Diagnosis | The config template contained `<YOUR-ENABLE-PASSWORD>` and was pasted unchanged |
| Root cause | The placeholder text itself became the enable secret and admin password |
| Fix | Logged in with the literal placeholder, then set new passwords with `enable secret` and `username admin secret` |
| Lesson | Replace every placeholder before pasting a template; keep credentials out of the repo |

---

## Entry 4: SSH login rejected, then connection refused

| Field | Detail |
|---|---|
| Phase | 4 (ACL + hardening) |
| Symptom | `ssh -l admin` returned "Login invalid" three times, then "Connection refused" on R1; SW1 rejected the same password |
| Diagnosis | The enable secret was used at the SSH password prompt; three failures within 30 s triggered `login block-for` quiet mode. SW1 still had its original local user because credentials are per device |
| Root cause | SSH login uses the local `username admin` password, not the enable secret, and each device keeps its own user database |
| Fix | Reset `username admin secret` and `enable secret` from the console on R1, SW1 and SW2, waited out the 60 s block, saved with `write memory` |
| Lesson | Quiet mode confirms `login block-for` works. Changing a user on one device does not change the others; at scale this is why centralized AAA (TACACS+/RADIUS) is used |

---

## Entry 5: Port security violation did not trigger

| Field | Detail |
|---|---|
| Phase | 4 (ACL + hardening) |
| Symptom | Violation test on Fa0/2 showed `Secure-up` and violation count 0; the swapped-in rogue PC could not ping the gateway |
| Diagnosis | `show port-security address` listed 0030.A33D.E7D0 on Fa0/2; `ipconfig /all` on the rogue PC showed the same MAC (and a /8 mask instead of /24) |
| Root cause | The rogue PC was the first device to send traffic on Fa0/2, so sticky learning locked its MAC instead of the legitimate PC's |
| Fix | Reconnected the legitimate PC, removed the learned sticky MAC and re-applied port security on Fa0/2, confirmed 0001.C717.2CD5 with `show port-security address`, then repeated the test: Fa0/2 went `Secure-shutdown` with violation count 1 |
| Lesson | Sticky learning is first-come, first-served: verify the learned MAC before testing violations. After a violation, the rogue's 100% ping loss is the expected result |