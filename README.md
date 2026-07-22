# Azure CSPM with Microsoft Defender for Cloud

![Azure](https://img.shields.io/badge/Azure-Cloud%20Security-0078D4?style=for-the-badge&logo=microsoftazure)
![Status](https://img.shields.io/badge/Status-Complete-brightgreen?style=for-the-badge)

## Overview
This project implements **Cloud Security Posture Management (CSPM)** using 
Microsoft Defender for Cloud on an existing Azure environment built across 
Projects 1–5 of this portfolio. CSPM continuously monitors cloud configurations 
against security best practices, surfacing misconfigurations before attackers 
find them.

The core question CSPM answers: *Is my cloud environment actually configured 
securely, or has it drifted from best practices over time?*

## Environment
This project assesses and remediates the existing `rg-zerotrust-security` 
resource group built in prior projects:

Azure Subscription 1 (fbabce0f-708b-4bd4-9fbb-dd0da8b958c8)
└── rg-zerotrust-security
├── vnet-zerotrust (10.0.0.0/16)
│   ├── snet-frontend  (10.0.1.0/24)
│   ├── snet-backend   (10.0.2.0/24)
│   └── snet-management (10.0.3.0/24)
├── stzerotrust1234 (Storage Account)
└── law-sentinel-project (Log Analytics Workspace)

## Baseline Assessment

### Secure Score at Start: 50%
- 6 assessed resources
- 2 unhealthy resources
- 9 recommendations (all "Not evaluated")
- Microsoft Cloud Security Benchmark: 54/63 controls passed
- Workload protection coverage: 0% (free CSPM tier — intentional)

### Findings Identified

| Finding | Resource | Severity | Status |
|---------|----------|----------|--------|
| Subscriptions should have a security contact email | Subscription | Medium | Unhealthy |
| More than one owner should be assigned | Subscription | Medium | Unhealthy |
| Storage accounts should restrict network access via VNet rules | stzerotrust1234 | Medium | Unhealthy |
| Storage accounts should prevent shared key access | stzerotrust1234 | Medium | Unhealthy |

## Remediations

### Remediation 1 — Security Contact Email
**Finding:** No security contact email configured on the subscription.

**Why it matters:** Without a security contact, Defender for Cloud has no way 
to alert the right person when high-severity threats are detected. In enterprise 
environments this is a critical governance gap — alerts go nowhere.

**Fix:** Configured `fritzekaney@hotmail.se` as security contact with Owner 
role notifications and High severity alert emails enabled.

![Security Contact](screenshots/01-security-contact-email.png)

---

### Remediation 2 — Second Subscription Owner
**Finding:** Only one Owner assigned to the subscription creates a single point 
of failure for administrative access.

**Why it matters:** If the sole Owner account is compromised, locked out, or 
unavailable, nobody can manage the subscription. Microsoft recommends at least 
two owners for resilience.

**Fix:** Assigned `testuser@fritzlearninghotmail.onmicrosoft.com` (TestUser-Security) 
as a second Owner with restricted delegation condition — cannot assign other 
privileged roles (Owner, UAA, RBAC).

![Second Owner Assigned](screenshots/02-second-owner-assigned.png)
![Role Assignments Confirmed](screenshots/02b-second-owner-role-assignments.png)

---

### Remediation 3 — Storage Network Access Restriction
**Finding:** `stzerotrust1234` accepted connections from all public networks 
despite having a Private Endpoint configured.

**Why it matters:** Even with a Private Endpoint, leaving public network access 
unrestricted means any attacker who discovers the storage endpoint can attempt 
access. Network-level restriction is defense in depth — it limits the attack 
surface to known, trusted networks only.

**Fix:** Changed public network access scope from "All networks" to "Selected 
networks," added `vnet-zerotrust/snet-backend` via service endpoint.

**Real-world blocker encountered:** The `Require-Environment-Tag` Azure Policy 
assignment had its description labeled "Audit resources missing the Environment 
tag" in the portal summary — but the actual policy definition enforced a hard 
**Deny** effect. This blocked both the VNet tagging operation and the service 
endpoint enablement.

**Resolution workflow:**
1. Identified the Deny effect by inspecting the raw policy definition JSON
2. Set policy enforcement to `DoNotEnforce` temporarily
3. Applied `Environment: Production` tag to `vnet-zerotrust`
4. Successfully enabled Microsoft.Storage service endpoint on `snet-backend`
5. Added VNet rule to storage account networking
6. Re-enabled policy enforcement

**Key learning:** Portal UI descriptions can show "Audit" while the underlying 
policy definition enforces "Deny." Always verify the actual policy definition, 
not just the summary label. This is a real-world trap that affects remediation 
workflows.

![Storage Network Restricted](screenshots/03-storage-network-restricted.png)

---

### Remediation 4 — Disable Shared Key Access
**Finding:** Shared key access enabled on `stzerotrust1234`.

**Why it matters:** Shared Key authentication bypasses Azure's entire identity 
layer — no MFA, no Conditional Access, no RBAC. Anyone with the storage account 
key has unrestricted access regardless of what IAM controls are in place. This 
directly undermines the IAM hardening work from Project 2.

**Fix:** Disabled "Allow storage account key access" in storage account 
Configuration. All access must now go through Microsoft Entra ID with proper 
RBAC assignments.

---

## Secure Score Improvement

| Metric | Before | After |
|--------|--------|-------|
| Secure Score | 50% | Pending re-scan |
| Unhealthy Resources | 2 | 0 (expected) |
| Active Recommendations | 4 | 0 (expected) |

**Note on re-scan lag:** Defender for Cloud assessments run on a periodic cycle 
(typically 12–24 hours). Resource configurations were verified as correct at 
the Azure resource level immediately after remediation. Dashboard reflection 
of changes is expected within 24 hours of remediation completion. This lag is 
documented behavior, not an error — in enterprise environments, security 
engineers verify at the resource level first; the dashboard provides confirmation, 
not proof.

## Real-World Findings & Talking Points

### 1. Policy Effect Mismatch
The `Require-Environment-Tag` policy displayed "Audit" in the portal assignment 
summary but enforced a hard Deny at the definition level. This caused unexpected 
blocks on VNet tagging and service endpoint operations during CSPM remediation.

**Interview talking point:** "I discovered that Azure Policy portal summaries 
can misrepresent the actual enforcement effect. I diagnosed this by inspecting 
the raw policy definition JSON, confirmed the Deny effect, and used DoNotEnforce 
mode as a controlled unblock mechanism — then re-enabled enforcement after 
completing the change."

### 2. Shared Key vs. Entra ID Authentication
Shared key access is a legacy authentication method that predates identity-based 
cloud security. Disabling it forces all storage access through Entra ID, enabling 
full audit trails, MFA enforcement, and Conditional Access policies.

### 3. Service Endpoints vs. Private Endpoints
This project used a service endpoint to restrict storage access to `snet-backend`. 
The environment also has a Private Endpoint (from Project 1) — these serve 
different purposes: Private Endpoints eliminate public exposure entirely; 
service endpoints restrict which VNets can use the public endpoint. Both are 
valid controls at different security levels.

## MITRE ATT&CK Relevance

| Technique | ID | How These Controls Mitigate It |
|-----------|-----|-------------------------------|
| Valid Accounts | T1078 | Second owner + security contact ensures faster detection |
| Data from Cloud Storage | T1530 | VNet restriction + shared key disabled limits access paths |
| Exfiltration to Cloud Storage | T1567 | Network rules block unauthorized egress to storage |

## Technologies Used
- Microsoft Defender for Cloud (CSPM — Free Tier)
- Microsoft Cloud Security Benchmark
- Azure Policy (Deny effects, DoNotEnforce mode)
- Azure Storage Networking (VNet rules, service endpoints)
- Azure RBAC (Owner role assignment with conditions)
- Microsoft Entra ID

## Related Projects
- Project 1: [Zero Trust Network Security](https://github.com/fritzekane/azure-zerotrust-network-security)
- Project 2: [IAM Hardening with Microsoft Entra ID](https://github.com/fritzekane/azure-iam-entra-id)
- Project 3: [Security Monitoring with Microsoft Sentinel](https://github.com/fritzekane/azure-sentinel-siem)
- Project 4: [Compliance Automation with Azure Policy](https://github.com/fritzekane/azure-policy-compliance)
- Project 5: [Endpoint Security with Microsoft Intune](https://github.com/fritzekane/azure-intune-endpoint-security)

---
*Part of my Azure Cloud Security Portfolio — 7 hands-on projects demonstrating 
real-world security engineering skills.*
