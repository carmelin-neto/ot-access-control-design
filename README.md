# Access Control Design: Role-Based Access for a Segmented OT Network

**Type:** Access control design extending the network segmentation from [ot-purdue-model-mapping](https://github.com/carmelin-neto/ot-purdue-model-mapping) beyond network-layer ACLs into role-based access control (RBAC) — who can do what, not just which packets can reach where.

**Why this is a distinct project, not a rehash of Project 1:** A network ACL answers one question: can this IP reach that IP. It says nothing about *who* is sitting at a workstation, *what* they're allowed to do once they've reached a device, or *why* a technician's legitimate access shouldn't also mean unrestricted access. Real intrusions are frequently the result of a legitimate account being misused — the Verizon DBIR consistently finds credential misuse among the top attack patterns — which network segmentation alone does nothing to prevent. This project designs the layer that segmentation doesn't cover.

---

## The Gap This Closes

[ot-purdue-model-mapping](https://github.com/carmelin-neto/ot-purdue-model-mapping) proved that a router ACL blocks direct IT-to-OT traffic. It says nothing about what happens *after* traffic legitimately reaches the OT zone — for instance, once an authorized engineering workstation is inside the DMZ, does every user of that workstation have equal ability to write to the PLC, or only some? The existing network design has no answer to that question. This project supplies one.

---

## Role Definitions

Based on realistic OT/ICS staffing patterns, four roles are defined:

| Role | Description | Typical Access Need |
|---|---|---|
| **OT Operator** | Monitors process state during a shift, responds to alarms | Read process values, acknowledge alarms — no configuration changes |
| **OT Engineer** | Configures and tunes control logic, calibrates devices | Read + write to PLC registers, modify control logic |
| **IT Security Analyst** | Monitors detection tooling, investigates alerts | Read Suricata/Splunk logs, read network flow data — no OT device access at all |
| **Vendor/Maintenance** | Occasional third-party access for firmware updates, hardware service | Time-boxed write access to a single specific device, nothing else |

**Design principle applied: Principle of Least Privilege (PoLP).** Every role gets the minimum access needed for its function — not "OT staff get OT access" as a blanket grant, which is the default many smaller environments fall into and exactly the pattern this design avoids.

---

## Access Control Matrix

| Resource | OT Operator | OT Engineer | IT Security Analyst | Vendor/Maintenance |
|---|---|---|---|---|
| PLC — Read registers | ✅ | ✅ | ❌ (no direct OT access) | ✅ (scoped to assigned device only) |
| PLC — Write registers | ❌ | ✅ | ❌ | ✅ (time-boxed, assigned device only) |
| HMI — View | ✅ | ✅ | ❌ | ❌ |
| HMI — Configuration change | ❌ | ✅ | ❌ | ❌ |
| Suricata/Splunk logs | ❌ | ❌ | ✅ | ❌ |
| Network flow data | ❌ | ❌ | ✅ | ❌ |
| DMZ jump host | ❌ (not needed — operators work from OT zone directly) | ✅ (route to OT zone) | ✅ (route to monitoring) | ✅ (temporary account, expires) |

**Notable design decision: the IT Security Analyst role has zero direct OT device access.** This is intentional, not an oversight — a SOC analyst's job is to *observe* the environment through logs and network telemetry, not to *operate* within it. Granting monitoring staff operational write access would violate least privilege and expand the blast radius if that analyst's credentials were ever compromised, for no operational benefit.

---

## Why the Vendor/Maintenance Role Is the Highest-Risk Row

Third-party vendor access is a well-documented weak point in OT environments — the account exists specifically to be handed to someone outside the organization's normal personnel controls, often for a narrow, time-limited task. This design addresses it with three specific constraints, not just an access grant:

1. **Scoped to a single assigned device** — a vendor servicing the PLC has no path to the HMI, other PLCs, or any device outside their assignment
2. **Time-boxed** — the account is provisioned with an expiration, not left open indefinitely (this directly addresses the root cause identified in [ot-incident-response-scenario](https://github.com/carmelin-neto/ot-incident-response-scenario): a stale, forgotten access grant was the root cause of that incident's containment failure)
3. **Logged distinctly** — vendor account activity is tagged separately in Splunk so a vendor session doing something unexpected is immediately distinguishable from normal operator/engineer activity, rather than blending into general traffic

---

## How This Prevents Lateral Movement Even If Segmentation Is Bypassed

[ot-risk-assessment](https://github.com/carmelin-neto/ot-risk-assessment) scored IT-to-OT lateral movement as Medium residual risk (down from High) largely due to network segmentation — but explicitly noted that a single-layer control can fail. RBAC is the second layer that risk assessment called for:

**Scenario: an attacker compromises an OT Operator's credentials** (via phishing, credential reuse, or similar — not a network-layer bypass at all). Even with valid, authenticated access:
- They can read process values and acknowledge alarms
- They **cannot** write to PLC registers — the exact action [ics-modbus-attack-testing](https://github.com/carmelin-neto/ics-modbus-attack-testing) demonstrated as the critical exploit path
- The attack that worked in Project 3 (a Modbus register write) is blocked here not by network segmentation but by role permissions — a genuinely independent control, not a duplicate of the ACL

This is the point of defense-in-depth: two controls that fail independently rather than one control repeated twice.

---

## Limitations, Stated Honestly

This is a **design document, not a deployed and tested implementation** — unlike the other projects in this portfolio, there is no live enforcement mechanism (no Active Directory group policy, no PLC-native user account configuration) built and validated here. GRFICS's simulated PLC does not expose a realistic multi-user authentication system to implement this against. The role definitions, access matrix, and reasoning are real security design work; the *enforcement* of it remains a gap, and I'm naming that gap directly rather than implying otherwise.

**What a next iteration would need to close this gap:** a lab environment with an OT device or HMI supporting real user-level authentication (rather than GRFICS's single-credential PLC access model), against which this access matrix could actually be configured and tested — the same "don't just argue it, prove it" standard the rest of this portfolio holds itself to.

---

## Portfolio Connections

| This project | Relationship |
|---|---|
| [ot-purdue-model-mapping](https://github.com/carmelin-neto/ot-purdue-model-mapping) | The network segmentation this design adds a role-based layer on top of |
| [ics-modbus-attack-testing](https://github.com/carmelin-neto/ics-modbus-attack-testing) | The exact write-attack this design's OT Operator restriction blocks independently of network controls |
| [ot-risk-assessment](https://github.com/carmelin-neto/ot-risk-assessment) | Identified segmentation as a single-layer control; this design supplies the second layer |
| [ot-incident-response-scenario](https://github.com/carmelin-neto/ot-incident-response-scenario) | The stale-access-grant root cause this design's time-boxed vendor access directly addresses |
