# Meridian Health Partners: Non-Human Identity Governance Policy

| Field | Value |
|---|---|
| Policy ID | MHP-IAM-POL-007 |
| Owner | Identity & Access Management |
| Applies to | All non-human identities in Entra ID, Azure, Key Vault, and Active Directory |
| Control mapping | NIST 800-53 AC-2, AC-6, IA-4, IA-5 · PCI-DSS 4.0 Req 8.6 · SOX ITGC logical access · HIPAA 164.312(a) |

## 1. Purpose

Machine identities authenticate to systems holding PHI and payment data with no MFA, no manager, and no HR record to trigger offboarding. This policy defines how MHP inventories them, what "compliant" means, and what happens when an identity fails.

## 2. Scope

In scope:

| NHI type | Source of truth |
|---|---|
| App registrations and their service principals | Microsoft Graph |
| User-assigned managed identities | Azure Resource Manager |
| API keys and secrets | Azure Key Vault |
| Service accounts and gMSAs | Active Directory (meridian domain) |
| User-type accounts used as service accounts (`svc-*`) | Microsoft Graph |

Out of scope for findings: Microsoft first-party service principals (owner tenant `f8cdef31-a31e-4b4a-93e4-5f571e91255a`). They are inventoried for completeness but governed by Microsoft.

## 3. Ownership requirement

Every NHI must have a named human owner accountable for its access and its credentials.

- App registrations: at least one Entra owner assigned
- Managed identities and Key Vault secrets: `owner` tag set
- AD service accounts: owner recorded in the `description` attribute as `owner=<upn>`
- User-type service accounts: owner set as the account's `manager` in Entra

An NHI with no owner is treated as orphaned.

## 4. Rules

### 4.1 Least privilege (LP)

| Rule | Condition | Severity |
|---|---|---|
| NHI-LP-01 | App holds a high-risk Microsoft Graph application permission (`Directory.ReadWrite.All`, `RoleManagement.ReadWrite.Directory`, `AppRoleAssignment.ReadWrite.All`, `Application.ReadWrite.All`, `Mail.ReadWrite`, `Sites.FullControl.All`) | Critical |
| NHI-LP-02 | Service principal holds a privileged Entra directory role | Critical |
| NHI-LP-03 | Identity holds Owner, Contributor, or User Access Administrator at subscription or management group scope | High |
| NHI-LP-04 | AD service account is a member of Domain Admins, Enterprise Admins, Schema Admins, Administrators, or Account Operators | Critical |
| NHI-LP-05 | User-type service account can sign in interactively | High |

### 4.2 Credential hygiene (CR)

| Rule | Condition | Severity |
|---|---|---|
| NHI-CR-01 | Client secret lifetime exceeds 180 days | High |
| NHI-CR-02 | More than one active client secret on one app | Medium |
| NHI-CR-03 | Any credential expires within 30 days (rotation due) | Medium |
| NHI-CR-04 | Key Vault secret has no expiration date | High |
| NHI-CR-05 | AD service account has PasswordNeverExpires set (gMSAs exempt) | High |
| NHI-CR-06 | User-based AD account has a servicePrincipalName (Kerberoastable; use a gMSA) | High |
| NHI-CR-07 | AD service account password last set more than 365 days ago | Medium |

Preferred credentials, in order: managed identity, then certificate, then client secret. Secrets are a last resort.

### 4.3 Necessity (NC)

| Rule | Condition | Severity |
|---|---|---|
| NHI-NC-01 | No owner assigned (orphaned) | High |
| NHI-NC-02 | No sign-in activity in 90 days, or never signed in | Medium |
| NHI-NC-03 | Managed identity attached to no resource and holding no role assignments | Medium |
| NHI-NC-04 | User-based AD service account enabled but has never logged on (gMSAs exempt) | Medium |

Activity signal by plane:

| NHI type | Activity rule | Signal |
|---|---|---|
| App registrations, user-type service accounts | NC-02 | Entra sign-in activity |
| Managed identities | NC-03 | Resource attachment and role assignments |
| AD service accounts | NC-04 | `lastLogonTimestamp` |
| Key Vault secrets | None | No native usage signal; orphaned secrets are rotated and reassigned, not deleted |

Lab calibration: the lab estate is new, so NC-02 flags any identity with no recorded sign-in at all. NHIs meant to be in use are exercised once during seeding.

## 5. Risk scoring

| Severity | Points |
|---|---|
| Critical | 40 |
| High | 25 |
| Medium | 10 |

An NHI's risk score is the sum of its findings. Scores of 50 and above go to the IAM lead within 5 business days.

## 6. Governance decisions

Each NHI receives one decision. The first matching row applies.

| Decision | Trigger | SLA |
|---|---|---|
| Decommission | Orphaned (NC-01) and unused (NC-02, NC-03, or NC-04) | 14 days, with a 30-day soft-delete recovery window |
| Remediate | Any LP finding | 14 days for Critical, 30 days for High |
| Rotate | Any CR finding | 30 days |
| Retain | No findings; owner re-attests annually | 365 days |

## 7. Exceptions

Exceptions need written owner justification, IAM lead approval, a compensating control, and an expiry no longer than 90 days.
