# Test Plan & Results — Vitalis Pharma Prague HQ

All tests were run in Cisco Packet Tracer 9.0.1 on 2026-10-08.

## 1. Physical layer & switching

| # | Test | Command | Expected | Result |
|---|------|---------|----------|--------|
| 1.1 | All end devices are cabled to the planned ports | `show interfaces status` (PRG-SW1) | Fa0/1-2, 11-12, 21-22, 24 `connected` | ✅ Pass |
| 1.2 | VLANs exist and ports are assigned | `show vlan brief` (PRG-SW1) | VLAN 10/20/30/99 with planned port ranges | ✅ Pass |
| 1.3 | Uplink to router is a trunk | `show running-config` (PRG-SW1) | `Gi0/1` → `switchport mode trunk` | ✅ Pass |

## 2. Routing & addressing

| # | Test | Command | Expected | Result |
|---|------|---------|----------|--------|
| 2.1 | Router subinterfaces are up | `show ip interface brief` (PRG-R1) | Gi0/0/0 and .10/.20/.30/.99 `up/up` | ✅ Pass |
| 2.2 | DHCP hands out addresses after the excluded range | `ipconfig` (USER-PC1, USER-PC2) | 10.1.10.11, 10.1.10.12 | ✅ Pass |
| 2.3 | Static management address | IP Configuration (ADMIN-PC) | 10.1.99.10/24, GW 10.1.99.1 | ✅ Pass |

## 3. Connectivity (before ACL)

Source: USER-PC1 (10.1.10.11)

| # | Destination | Expected | Result |
|---|-------------|----------|--------|
| 3.1 | 10.1.10.1 (own gateway) | Reply | ✅ 4/4 |
| 3.2 | 10.1.10.12 (USER-PC2, same VLAN) | Reply | ✅ 4/4 |
| 3.3 | 10.1.20.11 (LAB-PC1) | Reply | ✅ after fix (see Troubleshooting) |
| 3.4 | 10.1.30.11 (GUEST-PC1) | Reply | ✅ 3/4 (first packet lost to ARP) |
| 3.5 | 10.1.99.10 (ADMIN-PC) | Reply | ✅ 3/4 (first packet lost to ARP) |

## 4. Security — guest isolation (ACL `GUEST-IN`)

Source: GUEST-PC1 (10.1.30.11)

| # | Destination | Expected | Result |
|---|-------------|----------|--------|
| 4.1 | 10.1.30.1 (own gateway) | Reply (rule 20) | ✅ 4/4 |
| 4.2 | 10.1.30.12 (GUEST-PC2, same VLAN) | Reply (switched, never reaches ACL) | ✅ 4/4 |
| 4.3 | 10.1.10.11 (USERS) | Blocked | ✅ Destination host unreachable |
| 4.4 | 10.1.20.11 (LAB) | Blocked | ✅ Destination host unreachable |
| 4.5 | 10.1.99.10 (MGMT) | Blocked | ✅ Destination host unreachable |
| 4.6 | USER-PC1 → 10.1.30.x | Fails: request arrives, but the guest's reply is dropped by the stateless ACL | ✅ Request timed out |
| 4.7 | ACL hit counters | `show access-lists` (PRG-R1) | Rule 20 and rule 30 counters increase | ✅ 4 / 15 matches |

## 5. Security — management access (SSH + `access-class 10`)

| # | Test | Expected | Result |
|---|------|----------|--------|
| 5.1 | ADMIN-PC → `ssh -l admin 10.1.99.2` | Login to PRG-SW1 | ✅ `PRG-SW1#` |
| 5.2 | ADMIN-PC → `ssh -l admin 10.1.99.1` | Login to PRG-R1 | ✅ `PRG-R1#` |
| 5.3 | Wrong password | Rejected | ✅ `% Login invalid` |
| 5.4 | USER-PC1 → `ssh -l admin 10.1.99.1` | Refused (not from MGMT) | ✅ `% Connection refused by remote host` |
| 5.5 | USER-PC1 → `ping 10.1.99.1` | Reply — `access-class` protects only the management plane, not transit traffic | ✅ 4/4 |

## Troubleshooting log

**Symptom:** USER-PC1 could reach GUEST and MGMT, but ping to LAB-PC1 (10.1.20.11) failed 0/4.

**Steps:**
1. Ping to other VLANs worked → the router and trunk were fine in general.
2. Ping to the LAB gateway 10.1.20.1 succeeded → subinterface `.20` and VLAN 20 on the trunk were fine.
3. The fault was therefore isolated to the end device.

**Root cause:** the LAB PCs had not been switched to DHCP, so they had no IP address.

**Fix:** enabled DHCP on the LAB PCs → LAB-PC1 received 10.1.20.11 (verified with `ipconfig`), ping succeeded.
