# Network Segmentation: a Practical Guide for Network Engineers

> **Repo path (local):** `C:\Users\Max\repos\linux-security-hardening\docs\diagrams`  
> **Purpose:** Markdown documentation explaining the concept of network segmentation, zone design, policies, and implementation (SMB/Enterprise/Cloud/OT), including verification templates and a traffic-matrix worksheet.

---

## Table of Contents (TOC)

- [1. Introduction](#1-introduction)
- [2. What is network segmentation](#2-what-is-network-segmentation)
- [3. Why network segmentation matters](#3-why-network-segmentation-matters)
- [4. How segmentation works: three primary goals](#4-how-segmentation-works-three-primary-goals)
- [5. When to segment a network](#5-when-to-segment-a-network)
- [6. Design principles](#6-design-principles)
- [7. Network segmentation policies](#7-network-segmentation-policies)
- [8. Types and methods of segmentation](#8-types-and-methods-of-segmentation)
- [9. Implementation playbook](#9-implementation-playbook)
- [10. Segmentation examples](#10-segmentation-examples)
- [11. How to test segmentation](#11-how-to-test-segmentation)
- [12. What a network segment diagram should show](#12-what-a-network-segment-diagram-should-show)
- [13. DLP + PCI: segmentation for compliance](#13-dlp--pci-segmentation-for-compliance)
- [14. Segmentation vs ZTNA (Zero Trust Network Access)](#14-segmentation-vs-ztna-zero-trust-network-access)
- [15. Segmentation vs microsegmentation](#15-segmentation-vs-microsegmentation)
- [16. Traffic matrix (template)](#16-traffic-matrix-template)
- [17. Enterprise / SMB / Cloud / OT guidance](#17-enterprise--smb--cloud--ot-guidance)
  - [17.1 Enterprise](#171-enterprise)
  - [17.2 SMB](#172-smb)
  - [17.3 Cloud](#173-cloud)
  - [17.4 OT/ICS](#174-otics)
- [18. Common mistakes and anti-patterns](#18-common-mistakes-and-anti-patterns)
- [19. Quick checklist](#19-quick-checklist)

---

## 1. Introduction

Network segmentation is one of the most effective architectural measures for reducing risk, improving service stability, and simplifying operations. Good segmentation is not “more VLANs”; it is clearly defined trust zones, controlled traffic flows between zones, and policies that can be verified and maintained over time.

---

## 2. What is network segmentation

**Network segmentation** is the intentional separation of a network into logical or physical segments (zones) so that traffic between segments is **controlled** by defined rules (firewalls, ACLs, SDN policies) rather than freely flowing across the entire environment.

In practice, a segment is a group of devices/systems that share a similar trust level and security requirements (e.g., user endpoints, a server segment, a management segment, a guest network, IoT).

---

## 3. Why network segmentation matters

Segmentation:

- reduces the attack surface and limits lateral movement
- contains incident spread (malware/ransomware)
- enables least-privilege access at the network level (who can talk to whom)
- improves stability and performance (less broadcast/multicast “noise”, clearer routing)
- supports compliance (e.g., PCI DSS) by isolating the CDE and narrowing audit scope

---

## 4. How segmentation works: three primary goals

1. **Isolation and damage containment**  
   An incident in one segment should not automatically become a whole-network incident.

2. **Traffic-flow control (policy enforcement)**  
   “East–west” and “north–south” traffic should pass through enforcement points where rules are applied, logged, and monitored.

3. **Operational clarity (operations and change management)**  
   Clear boundaries simplify troubleshooting, addressing plans, QoS, and change control.

---

## 5. When to segment a network

You should segment almost always—especially when you have:

- different device classes: users, servers, administration, IoT, video surveillance, printers
- critical systems: AD/DNS, ERP, finance, backup, virtualization
- guests and BYOD: guest Wi‑Fi must be isolated from internal networks
- third parties: vendor VPN, external support, partners
- OT/ICS: industrial networks (PLC/SCADA) require strict isolation and protocol control
- regulatory requirements: CDE isolation and PCI scope reduction

---

## 6. Design principles

These practical principles make segmentation maintainable:

- **Zones based on trust level—not the org chart**  
  Segments should reflect risk and requirements, not only organizational structure.

- **Deny-by-default between zones**  
  Allow only what is necessary and document *why*.

- **Minimal flows (least privilege on the network)**  
  Avoid “any-any”. Every flow has a purpose, an owner, and a review cadence.

- **Enforcement points in the right places**  
  Inter-VLAN routing without controls is not segmentation. Controls must be enforceable and verifiable (firewall/ACL/SDN).

- **Management-plane isolation**  
  Network management (SSH/HTTPS/SNMP/NETCONF) must be separated from user segments.

- **Auditability (provability)**  
  You must be able to prove controls are effective: logs, tests, reviews.

- **Automate where possible**  
  In dynamic environments (cloud, VMs, containers), manual rules quickly become technical debt.

---

## 7. Network segmentation policies

A minimal policy set:

1. **Trust-zone model**  
   Define zones (Users, Servers, Mgmt, DMZ, Guest, IoT, Backup, CDE…).

2. **Allowed-traffic matrix**  
   Source → destination → protocol/port → purpose → service owner → change record.

3. **Exception management**  
   Every exception has an owner, risk statement, compensating controls, and an expiry/review date.

4. **Naming and documentation standards**  
   VLAN/VRF names, IP plan, rule descriptions, ticket/CR references.

5. **Rule lifecycle and review**  
   Periodic review and removal of unused/temporary rules.

---

## 8. Types and methods of segmentation

### 8.1 Segmentation types

- **Physical segmentation:** separate infrastructure (air-gap, dedicated switches/links)
- **Logical segmentation (L2/L3):** VLANs, subnets, routing, VRFs
- **Security zones:** firewall zones, internal firewalls (east–west)
- **Application segmentation:** 3‑tier (web/app/db), service zones
- **Microsegmentation:** workload-to-workload controls (VM/container)
- **Cloud segmentation:** VPC/VNet, security groups, NACLs, separated accounts/projects

### 8.2 Segmentation methods (techniques)

- VLAN + inter-VLAN controls (ACL/firewall)
- Subnetting and routing with filtering
- VRF (Virtual Routing and Forwarding) for strong isolation
- Firewall zones (perimeter and internal)
- ACLs on L3 switches/routers
- NAC / 802.1X (segmentation by user/device identity)
- SDN/overlay (VXLAN/EVPN) with centralized policies
- Host-based firewall/policies (workload segmentation)
- Container policies (e.g., Kubernetes NetworkPolicy)

---

## 9. Implementation playbook

### 9.1 Preparation (do not skip)

- inventory of devices and services (at minimum, a realistic list)
- data classification (public/internal/confidential/PCI)
- flow identification (who must talk to whom)

**Output:** an initial traffic matrix (template in section 16).

### 9.2 Design

- define zones + IP plan (VLAN IDs, subnets, gateways, DHCP/DNS)
- choose enforcement points (firewall/ACL/SDN)
- define the flow matrix and a deny-by-default posture

### 9.3 Implement controls

- create VLANs/subnets/VRFs
- routing (static/OSPF/BGP) + route filtering where required
- firewall/ACL rules: minimal, explicit, with logging where appropriate
- isolate the management plane (Mgmt segment)

### 9.4 Phased rollout

- start with a “pilot” segment (Guest or IoT), then servers, then users
- monitor blocked-traffic logs and refine the flow matrix

### 9.5 Documentation and maintenance

- diagrams, flow matrix, change procedure, test plan
- periodic rule reviews and removal of stale exceptions

---

## 10. Segmentation examples

### 10.1 SMB example (most common)

| Zone | VLAN | IP range | Example devices | Typical rules |
|---|---:|---|---|---|
| Users | 10 | 10.10.10.0/24 | endpoints | only required ports to servers |
| Servers | 20 | 10.10.20.0/24 | AD/DNS, file, app | admin only from Mgmt; limit east–west |
| Mgmt | 30 | 10.10.30.0/24 | admin workstations, NMS | SSH/HTTPS/SNMP only from Mgmt |
| Guest | 40 | 10.10.40.0/24 | guests | internet only |
| IoT/Printers | 50 | 10.10.50.0/24 | printers/cameras | Users → printer ports; block the rest |
| DMZ | 60 | 10.10.60.0/24 | reverse proxy/web | DMZ → internal strictly limited |
| Backup | 70 | 10.10.70.0/24 | backup repo | allow only required backup ports |

### 10.2 Data center (3-tier)

- **DMZ/Web** → **App** → **DB**
- The web tier must not access the DB directly; only the app tier talks to the DB.

### 10.3 OT/ICS concept

- IT zone, OT zone, Jump zone, Monitoring zone  
- allow only required protocols and only to specific destinations

---

## 11. How to test segmentation

### 11.1 Functional testing

- verify DNS/DHCP per segment
- test ports (e.g., `nc`, `curl`, `Test-NetConnection`)
- verify routing and gateways

### 11.2 Security testing (ensure blocked traffic stays blocked)

- scan from a “lower-trust” zone to a “higher-trust” zone and expect **deny**
- tcpdump/Wireshark at enforcement points
- review firewall logs (drop/deny events)
- simulate lateral movement and verify containment

### 11.3 Continuous testing and review

- after every policy change
- periodically (e.g., quarterly) to reduce technical debt

---

## 12. What a network segment diagram should show

A good segmentation diagram should show:

- zones/segments and their trust level
- VLAN IDs / subnets / VRFs (if used)
- enforcement points: firewalls, L3 gateways, NAC, proxies, DLP
- direction and type of flows (e.g., Users → App: TCP 443)
- critical services and ownership (team/responsible role)
- external links: internet, VPN, partners, cloud

**ASCII concept (minimal):**
```text
[Internet]
    |
 [Perimeter Firewall]
    |-------------------|
   DMZ               VPN/Remote
    |                   |
[Reverse Proxy]     [ZTNA/VPN GW]
    |
 [Internal firewall / zones]
  |     |       |        |
Users  Servers  Mgmt     IoT
```

---

## 13. DLP + PCI: segmentation for compliance

### 13.1 DLP (Data Loss Prevention)

DLP is a set of processes and tools that detect and/or prevent exfiltration of sensitive data (email, web uploads, cloud sync, USB, printing). Segmentation supports DLP because it:

- narrows where sensitive data is allowed to exist (e.g., only in the CDE or a dedicated “sensitive” zone)
- introduces choke points (proxy/egress firewall) where it is realistic to monitor outbound traffic
- simplifies rules (e.g., the CDE has no direct internet access except what is explicitly allowed)

### 13.2 PCI: compliance, CDE, and what qualifies as “PCI”

In PCI DSS contexts, segmentation is often used to isolate the **CDE (Cardholder Data Environment)** and reduce the compliance scope. In practice:

- the CDE includes systems that **store, process, or transmit** card data
- anything with **unrestricted connectivity** to the CDE can become in-scope
- “PCI data” typically includes the PAN (card number) and other elements of *cardholder data*, while *sensitive authentication data* (e.g., CVV/PIN) is treated especially strictly

> **Note:** If you use segmentation to reduce PCI scope, you must be able to prove that your segmentation controls are effective (testing + documentation).

---

## 14. Segmentation vs ZTNA (Zero Trust Network Access)

**Segmentation** controls flows between zones (boundaries and policies at the network level).  
**ZTNA** focuses on application access based on identity and context (who you are, device state, per-session policy), without implicit trust in an “internal network”.

In practice, the best result is a **combination**: ZTNA for “who/under what conditions”, segmentation for “where traffic is allowed to flow” and to limit blast radius.

### Quick comparison

| Aspect | Network segmentation | ZTNA |
|---|---|---|
| Control basis | zones, IPs, ports, protocols | identity, context, device state |
| Typical use | internal isolation and flow control | application access (especially remote/BYOD) |
| Strength | reduces blast radius | fine-grained per-session access |
| Risk | “any-any” exceptions undermine security | depends on identity stack/integrations |

---

## 15. Segmentation vs microsegmentation

- **Segmentation (classic):** coarse zones (Users/Servers/DMZ), enforced at firewall/L3 boundaries.
- **Microsegmentation:** fine-grained workload-to-workload controls (VM/container), often via SDN or host-based policies.

Microsegmentation is most valuable in dynamic environments (many services, frequent changes) and where “east–west” risk is high.

### When microsegmentation helps most

- large data centers or cloud environments with many services
- strong “service-to-service” communication control requirements
- automated policy management (IaC/SDN) and good flow visibility

---

## 16. Traffic matrix (template)

You can copy and fill in this table directly. Recommendation: every row should have a business justification, an owner, and a change reference (ticket/CR).

| ID | Source zone/segment | Destination zone/segment | Source (IP/CIDR/object) | Destination (IP/CIDR/object) | Protocol | Port(s) | Direction | Purpose / application | AuthN/AuthZ (if applicable) | Enforcement point (FW/ACL/SG) | Log (Y/N) | Criticality (L/M/H) | Owner | Ticket/CR | Status (Plan/Test/Prod) | Notes |
|---:|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 1 | Users | Servers | 10.10.10.0/24 | 10.10.20.10 | TCP | 443 | -> | Web access to application | SSO/LDAP | Internal FW | Y | M | App Team | CHG- | Plan |  |
| 2 | Mgmt | Network Devices | 10.10.30.0/24 | mgmt-objects | TCP | 22,443 | -> | Device administration | Admin RBAC | Mgmt FW/ACL | Y | H | NetOps | CHG- | Plan |  |
| 3 | Servers | Backup | 10.10.20.0/24 | 10.10.70.10 | TCP | (fill in) | -> | Backup replication | mTLS/Keys | Internal FW | Y | H | Infra | CHG- | Plan |  |

---

## 17. Enterprise / SMB / Cloud / OT guidance

### 17.1 Enterprise

**Goals:** scalability, standardization, multiple domains, complex flows, auditability.

Recommendations:
- firewall zones + VRF for strong isolation (e.g., prod/dev, tenants, CDE)
- centralized policy management (*policy-as-code* where possible)
- NAC/802.1X (dynamic policies, posture checks)
- logging + SIEM integration; segmentation controls as guardrails
- regular testing and reviews (quarterly or after changes)

### 17.2 SMB

**Goals:** simplicity, small team, quick maintenance.

Recommendations:
- 5–8 core zones (Users/Servers/Mgmt/Guest/IoT/DMZ/Backup)
- inter-VLAN controls (prefer enforcement via firewall)
- clear deny-by-default rules between Guest/IoT and internal networks
- document all exceptions and remove temporary rules regularly

### 17.3 Cloud

**Goals:** segmentation by project/account, identity-first control, ephemeral workloads.

Recommendations:
- separate accounts/projects for prod vs dev (where possible)
- VPC/VNet segmentation + routing control
- Security Groups / NSGs as the primary east–west control
- NACLs as an additional stateless layer where it makes sense
- private connectivity (PrivateLink/peering) with minimal routing
- automation: IaC (Terraform), policy guardrails, continuous compliance

### 17.4 OT/ICS

**Goals:** security + availability, controlled protocols, minimal change.

Recommendations:
- strict IT/OT separation + a controlled jump-host zone
- whitelist protocols and destinations (minimum required set)
- passive monitoring where possible; changes via controlled process
- backup/restore plan and an OT-specific incident plan
- physical segmentation or strong logical isolation when risk justifies it

---

## 18. Common mistakes and anti-patterns

- “We have VLANs” without inter-VLAN controls (in practice: no segmentation)
- “any-any” exceptions that become permanent
- Mgmt is not isolated (managing from the Users segment)
- rules without an owner or a purpose
- no periodic review (rules accumulate)
- segmentation “on paper” without tests and evidence

---

## 19. Quick checklist

- [ ] zones and trust levels are defined
- [ ] deny-by-default between zones
- [ ] traffic matrix is completed and approved
- [ ] Mgmt segment is isolated and strictly controlled
- [ ] enforcement points log relevant allow/deny events
- [ ] segmentation diagram + IP plan are up to date
- [ ] testing: functional + security + periodic
- [ ] exceptions have an owner, an expiry/review date, and a review process
