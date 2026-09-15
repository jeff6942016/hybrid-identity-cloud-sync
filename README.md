# Hybrid Identity Lab: Active Directory to Microsoft Entra ID with Cloud Sync

> A hybrid identity environment that synchronizes an on-premises Active Directory
> into Microsoft Entra ID using Microsoft Entra Cloud Sync with password hash sync,
> demonstrating the modern, lightweight path for extending an on-prem directory to
> the cloud. Built on a self-hosted domain controller syncing into a live Entra
> tenant.

![Windows Server](https://img.shields.io/badge/Windows%20Server-AD%20DS-0078D6?logo=windows&logoColor=white)
![Entra ID](https://img.shields.io/badge/Microsoft%20Entra%20ID-Cloud%20Sync-0067b8)
![Focus](https://img.shields.io/badge/Focus-Hybrid%20Identity-success)

---

## Overview

Most enterprises are not cloud-only. They run an on-premises Active Directory that
predates their move to the cloud, and identities have to exist in both places at
once: on-prem for legacy apps and file shares, and in Microsoft Entra ID for
Microsoft 365 and SaaS. Hybrid identity is the discipline of keeping those two
directories in sync from a single authoritative source, so a user is provisioned,
changed, and deprovisioned in one place and the change flows everywhere.

This lab builds that pipeline with **Microsoft Entra Cloud Sync**, the current
recommended tool for new deployments. Cloud Sync uses a lightweight, auto-updating
provisioning agent with no on-premises SQL database and no heavy sync engine; the
synchronization logic runs in Microsoft's cloud. On-premises AD users are synced up
into an Entra tenant with **password hash sync**, so the same credential works in
both directories.

## Why Cloud Sync (not Connect Sync)

Microsoft's current guidance is that Cloud Sync is the recommended choice for new
projects, with the legacy Connect Sync retained only for complex scenarios such as
pass-through authentication, group writeback, or custom transformation rules.
Choosing Cloud Sync here reflects the direction Microsoft is steering all customers.

| | Cloud Sync | Connect Sync (legacy) |
|---|---|---|
| Agent | Lightweight, auto-updating | Heavy, manual updates |
| On-prem database | None | SQL / LocalDB |
| Recommended for | New deployments | Complex edge cases |
| Managed by | Microsoft cloud | On-prem server |

## Architecture

```mermaid
graph LR
    AD["On-prem Active Directory<br/>corp.jefflab"] -->|Provisioning Agent| SYNC["Entra Cloud Sync<br/>(cloud-hosted)"]
    SYNC -->|Password Hash Sync| ENTRA["Microsoft Entra ID<br/>jeffreylpfyahoo.onmicrosoft.com"]
    style AD fill:#bbf,stroke:#333,stroke-width:2px
    style SYNC fill:#bfb,stroke:#333,stroke-width:2px
    style ENTRA fill:#fbf,stroke:#333,stroke-width:2px
```

| Component | Role |
|-----------|------|
| Windows Server (AD DS) | On-premises domain controller and authoritative source |
| Entra provisioning agent | Lightweight connector installed on the DC |
| Microsoft Entra Cloud Sync | Cloud-hosted synchronization service |
| Microsoft Entra ID | Target cloud directory |

---

## Build

Each stage below pairs the configuration step with the reason it matters, and shows
the process, not just the end state.

### 1. Domain controller foundation

A Windows Server VM is given a static IP (a DC's address cannot move, since it also
serves as the domain's DNS), then promoted to a domain controller for a new forest.

**Why it matters:** The domain controller is the authoritative source for the whole
hybrid pipeline. Every identity that syncs to the cloud originates here, so the
on-prem directory has to exist and be healthy before anything can flow upward.

![Static IP configured on the server](./screenshots/01-static-ip.png)

![AD DS role installation](./screenshots/02-adds-role-install.png)

![Promoting the server to a domain controller](./screenshots/03-dc-promotion.png)

![The promoted domain controller](./screenshots/04-dc-promoted.png)

### 2. Sync scope: OU, users, and UPN suffix

A dedicated organizational unit holds the users to be synced. Before creating them,
an alternative UPN suffix matching the verified Entra domain is added, so synced
users receive routable, sign-in-ready usernames rather than unusable internal ones.

**Why it matters:** Cloud Sync scopes by organizational unit, so a dedicated OU
gives precise control over exactly who syncs. The UPN suffix step is the common
pitfall: an on-prem `user@corp.jefflab` UPN is non-routable and will not work as a
cloud sign-in, so matching the UPN suffix to a verified Entra domain up front is
what makes the synced identities actually usable.

![Adding the verified-domain UPN suffix in AD Domains and Trusts](./screenshots/05-upn-suffix.png)

![Dedicated OU with the users to be synced](./screenshots/06-sync-users-ou.png)

![A test user with the correct routable UPN](./screenshots/07-user-upn.png)

### 3. Provisioning agent installation

The lightweight Entra provisioning agent is downloaded from the Entra admin center
and installed directly on the domain controller (a supported configuration). During
setup it creates a group managed service account (gMSA) to run its service.

**Why it matters:** The agent is the only on-premises footprint of Cloud Sync. It
makes outbound-only connections to Microsoft's cloud and auto-updates, which is what
makes Cloud Sync lightweight compared to the legacy sync engine. Running it under a
gMSA means its credentials are managed automatically by AD rather than being a
static password an administrator has to rotate.

![Downloading the provisioning agent from the Entra admin center](./screenshots/08-agent-download.png)

![Agent configuration wizard: selecting the Cloud Sync extension](./screenshots/09-agent-wizard.png)

![Providing domain credentials to create the gMSA](./screenshots/10-agent-gmsa.png)

![Agent registered and healthy in the portal](./screenshots/11-agent-healthy.png)

### 4. Cloud Sync configuration

A synchronization configuration is created in the Entra admin center: AD to Entra,
password hash sync enabled, scoped to the dedicated OU, then set to Enabled.

**Why it matters:** This is where the sync behavior is defined. Password hash sync
lets the same credential authenticate in both directories, and OU scoping enforces
that only intended users leave the on-prem boundary. Enabling it deliberately, after
scoping, mirrors how a real rollout is controlled rather than syncing an entire
directory blindly.

![Creating the AD-to-Entra configuration with password hash sync](./screenshots/12-cloud-sync-config.png)

![Scoping the configuration to the sync OU](./screenshots/13-scope-ou.png)

### 5. Provisioning and verification

Before enabling the full cycle, a single user is provisioned on demand to confirm
the pipeline. The scheduled sync then runs automatically, and the on-prem users
appear in Entra ID as directory-synced objects.

**Why it matters:** Provision on demand validates the whole chain against one
identity before it is trusted at scale, which is the safe way to test any sync.
The final state, on-prem users showing as synced in the cloud directory, is the
proof that hybrid identity is working: one authoritative source, two directories in
agreement.

![Provision on demand succeeding for a single user](./screenshots/14-provision-on-demand.png)

![Synced users in Entra ID, marked on-premises sync enabled](./screenshots/15-synced-users-entra.png)

The synced accounts appear alongside cloud-only users from the tenant, showing two
distinct identity sources coexisting in one directory, which is exactly what hybrid
identity means.

---

## Key Concepts Demonstrated

Hybrid identity architecture; on-premises Active Directory as an authoritative
source; Microsoft Entra Cloud Sync with a lightweight provisioning agent; password
hash synchronization; UPN routing and verified-domain alignment; OU-scoped sync;
group managed service accounts; and provision-on-demand validation.

## Defensive Value

- A single authoritative source for identity means a user is created, changed, and
  disabled in one place, closing the gap where an account lingers in one directory
  after being removed from another.
- OU-scoped synchronization enforces a deliberate boundary on which on-prem
  identities reach the cloud, rather than exposing an entire directory by default.
- Running the agent under a gMSA removes a static service-account password from the
  environment, one fewer credential for an attacker to harvest.

## Skills Demonstrated

- Hybrid identity design and Active Directory to Entra ID synchronization
- Windows Server AD DS: domain controller build, DNS, forest promotion
- Microsoft Entra Cloud Sync deployment and provisioning-agent installation
- Password hash sync and UPN/verified-domain configuration
- Scoped, controlled rollout with provision-on-demand testing

## Tech Stack

Windows Server (AD DS), Microsoft Entra ID, Microsoft Entra Cloud Sync, Microsoft
Entra provisioning agent.

## What I Learned

<!-- Note the real gotchas you hit: e.g. the UPN suffix step, IE Enhanced Security
blocking the agent sign-in, the gMSA credential prompt, static IP vs the VM's
network mode, or the sync cycle timing. A short honest paragraph here is one of the
strongest parts of the writeup. -->

## About

Built by Jeffrey Lam-Ping-Fong, a fourth-year Honours Bachelor of Information
Technology student specializing in cybersecurity, with a focus on identity and
access management.

- LinkedIn: https://www.linkedin.com/in/jeffrey-lam-ping-fong-07a649321
