# Multi-Cloud Landing Zone Guide — AWS, Azure & GCP

> A practical reference for designing and building a **recommended (enterprise-grade) landing zone** on AWS, Azure and Google Cloud — covering prerequisites, target architecture, **manual (console) setup**, **Terraform setup**, and **what happens when you add a new account / subscription / project later** (including how policies flow to it).

---

## Table of Contents

1. [What is a Landing Zone?](#1-what-is-a-landing-zone)
2. [Common Building Blocks (all clouds)](#2-common-building-blocks-all-clouds)
3. [Prerequisites Checklist](#3-prerequisites-checklist)
4. [Terminology Mapping Across Clouds](#4-terminology-mapping-across-clouds)
5. [AWS Landing Zone](#5-aws-landing-zone)
6. [Azure Landing Zone](#6-azure-landing-zone)
7. [GCP Landing Zone](#7-gcp-landing-zone)
8. [Adding a New Account / Subscription / Project After Setup](#8-adding-a-new-account--subscription--project-after-setup)
9. [How Policies Affect a New Account (Inheritance Deep-Dive)](#9-how-policies-affect-a-new-account-inheritance-deep-dive)
10. [Recommended Terraform Repository Layout](#10-recommended-terraform-repository-layout)
11. [CI/CD, State & Security for the Landing Zone Code](#11-cicd-state--security-for-the-landing-zone-code)
12. [IP Address Planning (Multi-Cloud)](#12-ip-address-planning-multi-cloud)
13. [Common Pitfalls](#13-common-pitfalls)
14. [Day-2 Operations](#14-day-2-operations)
15. [References](#15-references)

---

## 1. What is a Landing Zone?

A **landing zone** is a pre-configured, secure, scalable, multi-account (multi-subscription / multi-project) cloud environment that every workload "lands" into. It is built **once** by the platform team and then **reused** for every new application/team.

It answers these questions *before* the first workload arrives:

| Question | Landing zone answer |
|---|---|
| Who can log in, and how? | Central identity (SSO / Entra ID / Cloud Identity) + groups + least privilege |
| Where do resources live? | A resource hierarchy (OUs / Management Groups / Folders) |
| What is never allowed? | Preventive guardrails (SCP / Azure Policy deny / Org Policy) |
| What do we detect? | Detective guardrails (Config / Defender / SCC) |
| Where do logs go? | Central, immutable log archive |
| How do networks connect? | Hub-and-spoke / Transit / Shared VPC, central egress, hybrid connectivity |
| How is cost tracked? | Billing structure, tags/labels, budgets |
| How do we create the next account? | Account / subscription / project **vending** (ideally Terraform + PR) |

**Golden rule:** treat the landing zone as a **product** — versioned, in Git, deployed via pipeline, with owners and a change process.

---

## 2. Common Building Blocks (all clouds)

| # | Pillar | What you must design |
|---|---|---|
| 1 | **Identity & Access** | IdP (Entra ID / Okta / Google Workspace), SSO, MFA, groups → roles mapping, break-glass accounts, no long-lived keys |
| 2 | **Resource Hierarchy** | OU / MG / Folder tree, by *policy need* (prod vs non-prod, regulated vs not), **not** by org chart |
| 3 | **Guardrails – Preventive** | Deny regions, deny disabling logs, deny public IPs/buckets, enforce encryption, restrict root/owner |
| 4 | **Guardrails – Detective** | Compliance rules, CSPM, drift detection |
| 5 | **Logging & Audit** | Org-wide audit trail → dedicated log account/subscription/project, immutable storage, retention |
| 6 | **Security Services** | Threat detection (GuardDuty / Defender / SCC), vulnerability mgmt, central security account with delegated admin |
| 7 | **Networking** | IPAM, hub/transit, central egress & inspection, DNS (private zones + resolvers), hybrid (VPN / Direct Connect / ExpressRoute / Interconnect) |
| 8 | **Cost Management** | Billing hierarchy, mandatory tags/labels, budgets & alerts |
| 9 | **Vending** | Automated creation of new accounts with baseline applied |
| 10 | **IaC & Pipelines** | Terraform, remote state, OIDC-based CI, policy-as-code checks |

---

## 3. Prerequisites Checklist

### 3.1 Business / design decisions (decide BEFORE building)

- [ ] **Primary & DR regions** for each cloud (e.g., AWS `ap-south-1` + `ap-south-2`; Azure `centralindia` + `southindia`; GCP `asia-south1` + `asia-south2`)
- [ ] **Allowed regions** list (data residency / compliance — e.g., DPDP Act, ISO 27001, SOC 2, PCI)
- [ ] **Environment model**: `prod`, `non-prod` (`dev`, `qa`, `uat`), `sandbox`
- [ ] **Naming convention** (accounts, subscriptions, projects, resources)
- [ ] **Tagging / labelling standard** (`owner`, `cost-center`, `environment`, `application`, `data-classification`)
- [ ] **Network CIDR plan** — non-overlapping across *all three clouds* and on-prem
- [ ] **Hybrid connectivity**: VPN vs dedicated (Direct Connect / ExpressRoute / Interconnect)
- [ ] **Identity source**: single IdP federated to all clouds (recommended: Entra ID or Okta)
- [ ] **Log retention** (e.g., 1 year hot, 7 years archive)
- [ ] **Who owns what** (platform team, security team, app teams) → RACI

### 3.2 Technical prerequisites

| Item | AWS | Azure | GCP |
|---|---|---|---|
| Top-level container | A **new, clean management account** (payer) | An **Entra ID tenant** | A **Google Cloud Organization** (needs Cloud Identity / Workspace on a verified domain) |
| Billing | Payment method on management account | EA / MCA / CSP agreement with permission to create subscriptions | **Billing account** linked to the org |
| Unique emails | One unique email per account (use `+` aliases or a distribution list, e.g. `aws-prod-app1@company.com`) | Not needed per subscription | Not needed per project |
| Admin rights | Root + admin in mgmt account | **Global Admin** → elevate to **User Access Administrator at root `/`** (temporarily) | **Super Admin** (Workspace) + **Organization Administrator** |
| Groups | IAM Identity Center groups | Entra ID security groups | Cloud Identity groups (`gcp-org-admins`, `gcp-billing-admins`, `gcp-network-admins`, `gcp-security-admins`, `gcp-devops`…) |
| Tools | Terraform ≥ 1.6, AWS CLI v2 | Terraform ≥ 1.9, Azure CLI, (ALZ PowerShell module for accelerator) | Terraform ≥ 1.6, gcloud SDK |
| Git + CI | GitHub / GitLab / Azure DevOps with OIDC federation to each cloud | | |
| Quotas | Organizations account limit (default is low — raise via Service Quotas before mass vending) | Subscription creation limits per billing scope | Project creation quota per billing account / user |

---

## 4. Terminology Mapping Across Clouds

| Concept | AWS | Azure | GCP |
|---|---|---|---|
| Top of hierarchy | Organization (root) | Tenant Root Management Group | Organization node |
| Grouping layer | Organizational Unit (OU) | Management Group (MG) | Folder |
| Isolation / billing unit | **Account** | **Subscription** (+ Resource Groups) | **Project** |
| Preventive guardrail | SCP, RCP, Declarative policies, Control Tower *preventive* controls | Azure Policy (`Deny`, `DeployIfNotExists`, `Modify`) | Organization Policy constraints, IAM Deny policies, VPC Service Controls |
| Detective guardrail | AWS Config rules, CT *detective* controls, Security Hub | Azure Policy (`Audit`), Defender for Cloud | Security Command Center, Policy Intelligence |
| Identity | IAM Identity Center (SSO) | Entra ID + Azure RBAC | Cloud Identity + IAM |
| Central audit log | Org CloudTrail → Log Archive account | Activity Log diagnostic settings → Log Analytics / Storage | Aggregated org log sink → logging project |
| Threat detection | GuardDuty, Security Hub, Inspector | Defender for Cloud, Sentinel | Security Command Center |
| Hub network | Transit Gateway / Cloud WAN in Network account | Hub VNet or Virtual WAN in Connectivity subscription | Shared VPC host project / Network Connectivity Center |
| Account vending | Control Tower Account Factory / **AFT** | **Subscription vending** module | **Project Factory** module |
| Native LZ accelerator | **AWS Control Tower** (+ LZA as option) | **ALZ IaC Accelerator** (AVM modules) | **Google Cloud Setup** / **Fabric FAST** / **Example Foundation** |

---

## 5. AWS Landing Zone

### 5.1 Recommended approach

| Option | When to use |
|---|---|
| **AWS Control Tower + AFT (Account Factory for Terraform)** ✅ *recommended for Terraform shops* | Standard enterprise LZ, Terraform-first teams, GitOps account vending |
| Control Tower + CfCT (Customizations for Control Tower) | CloudFormation-first teams |
| Landing Zone Accelerator on AWS (LZA) | Highly regulated (gov, healthcare, finance) with heavy config-driven baselines |
| Pure Terraform (Organizations + SCPs, no Control Tower) | Full control needed, but you maintain *everything* yourself |

### 5.2 Target architecture (recommended OU structure)

```
Root
├── Security (OU)                 ← created by Control Tower
│   ├── Log Archive               ← org CloudTrail, Config, VPC flow logs (S3, Object Lock)
│   └── Audit (Security Tooling)  ← delegated admin: GuardDuty, Security Hub, Config aggregator, Inspector, Macie
├── Infrastructure (OU)
│   ├── Network                   ← Transit Gateway, central egress (NAT/Network Firewall), Route 53 Resolver, IPAM
│   └── Shared Services           ← CI/CD runners, artifact repos, AD/DNS, monitoring tools
├── AFT-Management (OU)
│   └── AFT Management            ← AFT pipelines (CodePipeline/CodeBuild, Step Functions, DynamoDB)
├── Workloads (OU)
│   ├── Prod (OU)                 ← app-a-prod, app-b-prod
│   └── NonProd (OU)              ← app-a-dev, app-a-qa, app-a-uat
├── Sandbox (OU)                  ← experimentation, budget-capped, no on-prem connectivity
├── Policy-Staging (OU)           ← test new SCPs here first
└── Suspended (OU)                ← closed / quarantined accounts, deny-all SCP
```

> **Management account** = billing + Organizations + Control Tower only. **No workloads.** SCPs do **not** apply to the management account, so keep it minimal and locked down.

### 5.3 Manual (Console) Setup — Step by Step

**Step 0 — Prepare the management account**
1. Create a fresh AWS account (this becomes the management/payer account).
2. Secure root: hardware MFA, strong password, remove root access keys, set alternate contacts (billing/ops/security).
3. Create an emergency **break-glass IAM user/role** (MFA, stored in a vault, alerting on use).
4. Prepare **2 unique email addresses** for Log Archive and Audit accounts.

**Step 1 — Set up AWS Control Tower**
1. Console → **AWS Control Tower** → *Set up landing zone*.
2. **Home region** (e.g., `ap-south-1`) — *cannot be changed later*.
3. **Governed regions** — select only allowed regions; enable **Region deny control** to block the rest.
4. Foundational OU name (`Security`) and an additional OU (e.g., `Sandbox`).
5. Log Archive & Audit accounts (new emails or existing accounts).
6. **AWS CloudTrail**: org-level trail (on). **AWS Config**: enabled. Configure log retention (e.g., 365 days / 7 years).
7. **KMS encryption** (recommended): create a customer-managed key for CloudTrail/Config.
8. **IAM Identity Center**: let Control Tower manage it, or self-manage (recommended if you already use Entra/Okta).
9. Launch — takes ~60 minutes.

**Step 2 — Identity (IAM Identity Center)**
1. Change identity source to **external IdP** (Entra ID / Okta) via SAML + **SCIM** for user/group sync.
2. Create **permission sets**: `AdministratorAccess` (break-glass/platform), `PowerUser`, `ReadOnly`, `NetworkAdmin`, `SecurityAudit`, `Billing`, custom app-team sets.
3. Assign groups → permission sets → accounts.

**Step 3 — Create OUs and register them**
1. Organizations → create OUs: `Infrastructure`, `Workloads/Prod`, `Workloads/NonProd`, `Policy-Staging`, `Suspended`, `AFT-Management`.
2. Control Tower → *Organization* → **Register OU** for each (so controls & baseline apply).

**Step 4 — Enable controls (guardrails)**
- Control Tower → *Control library* → enable on OUs. Recommended: all **Mandatory** + **Strongly recommended** controls, plus selected **Elective** ones (e.g., disallow public S3, require MFA for root, disallow changes to log archive bucket, deny creation of IAM users with access keys).
- Add **custom SCPs** (see 5.5) via Organizations → Policies → Service control policies.

**Step 5 — Security services (from the Audit account as delegated admin)**
1. From management account: designate **Audit** account as delegated admin for GuardDuty, Security Hub, Inspector, Macie, IAM Access Analyzer, Detective, Firewall Manager.
2. In Audit account: enable **auto-enable for new accounts** in each service (critical for future accounts!).
3. Security Hub: enable *AWS Foundational Security Best Practices* + *CIS* standards, **central configuration** at the org level.

**Step 6 — Networking (Network account)**
1. **VPC IPAM** (delegated to Network account) with pools per region/environment.
2. **Transit Gateway**, share via **AWS RAM** to the Organization or specific OUs.
3. Central **egress VPC** (NAT Gateway + optional AWS Network Firewall) and **inspection VPC**.
4. **Route 53 Resolver** endpoints & rules (shared via RAM) for hybrid DNS.
5. **Site-to-Site VPN / Direct Connect** to on-prem.

**Step 7 — Cost governance**
- Enable **Cost Explorer**, **Cost and Usage Report (CUR 2.0 / Data Exports)**, activate **cost allocation tags**, create **AWS Budgets** per account, enable **Cost Anomaly Detection**.
- Use **Tag Policies** (Organizations) to enforce tag key/value format.

**Step 8 — Account Factory**
- Control Tower → *Account Factory* → configure network settings (or disable VPC creation if you vend VPCs via Terraform/IPAM) → *Create account*.

### 5.4 Terraform Setup

#### Layered approach

| Layer | What | Tool |
|---|---|---|
| L0 | Management account bootstrap (state bucket, OIDC role) | Terraform (local state, then migrate) |
| L1 | Control Tower landing zone | `aws_controltower_landing_zone` *or* console (one-time) |
| L2 | OUs, SCPs, RCPs, tag policies, controls | `aws_organizations_*`, `aws_controltower_control` |
| L3 | AFT deployment | `aws-ia/control_tower_account_factory/aws` |
| L4 | Account requests + global/account customizations | AFT repos (GitOps) |
| L5 | Shared network, security delegated admins | Terraform in Network & Audit accounts |

#### L0 — State backend (in management account or a dedicated "terraform-state" account)

```hcl
# backend.tf
terraform {
  required_version = ">= 1.6"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" } # or ~> 6.0 – pin to what you validate
  }
  backend "s3" {
    bucket       = "org-tfstate-<mgmt-account-id>-ap-south-1"
    key          = "landing-zone/organization.tfstate"
    region       = "ap-south-1"
    encrypt      = true
    use_lockfile = true   # S3 native locking (Terraform >= 1.10); else use dynamodb_table
  }
}
```

#### L1 — Control Tower landing zone (Terraform)

> Log Archive and Audit accounts must already exist in the Organization when using the API-based setup.

```hcl
resource "aws_controltower_landing_zone" "this" {
  version = "3.3" # check latest supported LZ version
  manifest_json = jsonencode({
    governedRegions = ["ap-south-1", "ap-south-2"]
    organizationStructure = {
      security = { name = "Security" }
      sandbox  = { name = "Sandbox" }
    }
    centralizedLogging = {
      accountId = var.log_archive_account_id
      configurations = {
        loggingBucket       = { retentionDays = 365 }
        accessLoggingBucket = { retentionDays = 3650 }
        kmsKeyArn           = aws_kms_key.ct.arn
      }
      enabled = true
    }
    securityRoles = { accountId = var.audit_account_id }
    accessManagement = { enabled = true }
  })
}
```

#### L2 — OUs, SCPs, Controls

```hcl
data "aws_organizations_organization" "org" {}

locals { root_id = data.aws_organizations_organization.org.roots[0].id }

resource "aws_organizations_organizational_unit" "infrastructure" {
  name      = "Infrastructure"
  parent_id = local.root_id
}
resource "aws_organizations_organizational_unit" "workloads" {
  name      = "Workloads"
  parent_id = local.root_id
}
resource "aws_organizations_organizational_unit" "prod" {
  name      = "Prod"
  parent_id = aws_organizations_organizational_unit.workloads.id
}
resource "aws_organizations_organizational_unit" "nonprod" {
  name      = "NonProd"
  parent_id = aws_organizations_organizational_unit.workloads.id
}
resource "aws_organizations_organizational_unit" "suspended" {
  name      = "Suspended"
  parent_id = local.root_id
}

# ---- Example SCP: protect baseline + deny leaving org + deny root usage ----
data "aws_iam_policy_document" "baseline_scp" {
  statement {
    sid       = "DenyLeaveOrg"
    effect    = "Deny"
    actions   = ["organizations:LeaveOrganization"]
    resources = ["*"]
  }
  statement {
    sid    = "DenyRootUser"
    effect = "Deny"
    actions   = ["*"]
    resources = ["*"]
    condition {
      test     = "StringLike"
      variable = "aws:PrincipalArn"
      values   = ["arn:aws:iam::*:root"]
    }
  }
  statement {
    sid    = "ProtectSecurityServices"
    effect = "Deny"
    actions = [
      "guardduty:DeleteDetector", "guardduty:DisassociateFromMasterAccount",
      "securityhub:DisableSecurityHub", "access-analyzer:DeleteAnalyzer",
      "ec2:DisableEbsEncryptionByDefault",
    ]
    resources = ["*"]
    condition {
      test     = "ArnNotLike"
      variable = "aws:PrincipalArn"
      values   = ["arn:aws:iam::*:role/AWSControlTowerExecution", "arn:aws:iam::*:role/OrgPlatformAdmin"]
    }
  }
}

resource "aws_organizations_policy" "baseline" {
  name    = "scp-baseline-guardrails"
  type    = "SERVICE_CONTROL_POLICY"
  content = data.aws_iam_policy_document.baseline_scp.json
}

resource "aws_organizations_policy_attachment" "baseline_workloads" {
  policy_id = aws_organizations_policy.baseline.id
  target_id = aws_organizations_organizational_unit.workloads.id
}

# ---- Deny-all for Suspended OU ----
resource "aws_organizations_policy" "deny_all" {
  name    = "scp-deny-all"
  type    = "SERVICE_CONTROL_POLICY"
  content = jsonencode({
    Version   = "2012-10-17"
    Statement = [{ Effect = "Deny", Action = "*", Resource = "*" }]
  })
}
resource "aws_organizations_policy_attachment" "suspended" {
  policy_id = aws_organizations_policy.deny_all.id
  target_id = aws_organizations_organizational_unit.suspended.id
}

# ---- Tag policy ----
resource "aws_organizations_policy" "tags" {
  name = "tag-policy-mandatory"
  type = "TAG_POLICY"
  content = jsonencode({
    tags = {
      environment = {
        tag_key   = { "@@assign" = "environment" }
        tag_value = { "@@assign" = ["prod", "dev", "qa", "uat", "sandbox"] }
      }
    }
  })
}

# ---- Control Tower control on an OU ----
resource "aws_controltower_control" "deny_public_s3" {
  control_identifier = "arn:aws:controltower:ap-south-1::control/<CONTROL_ID>" # from Control Catalog
  target_identifier  = aws_organizations_organizational_unit.prod.arn
}
```

> Other useful policy types to manage the same way: **RCPs** (`RESOURCE_CONTROL_POLICY`) to stop external access to S3/KMS/STS/SQS/Secrets Manager, **Declarative policies** (e.g., block public AMIs, enforce IMDSv2), **Backup policies**, **AI services opt-out policies**.

#### L3 — Deploy AFT

```hcl
module "aft" {
  source  = "aws-ia/control_tower_account_factory/aws"
  version = "~> 1.x" # pin to a tested release

  ct_management_account_id    = var.ct_management_account_id
  log_archive_account_id      = var.log_archive_account_id
  audit_account_id            = var.audit_account_id
  aft_management_account_id   = var.aft_management_account_id
  ct_home_region              = "ap-south-1"
  tf_backend_secondary_region = "ap-south-2"

  # VCS – GitHub via CodeConnections (or CodeCommit / GitLab / Bitbucket)
  vcs_provider                                  = "github"
  account_request_repo_name                     = "my-org/aft-account-request"
  global_customizations_repo_name               = "my-org/aft-global-customizations"
  account_customizations_repo_name              = "my-org/aft-account-customizations"
  account_provisioning_customizations_repo_name = "my-org/aft-account-provisioning-customizations"

  aft_feature_delete_default_vpcs_enabled = true
  aft_feature_cloudtrail_data_events      = false
  aft_feature_enterprise_support          = false
  terraform_distribution                  = "oss"
}
```

AFT creates 4 Git repos (you own them):

| Repo | Purpose |
|---|---|
| `aft-account-request` | One `module` block per account → creates account |
| `aft-global-customizations` | Terraform run in **every** account (baseline: IAM roles, EBS default encryption, S3 block public access, password policy, budgets) |
| `aft-account-customizations` | Terraform per **customization profile** (e.g., `prod-baseline`, `sandbox-baseline`) |
| `aft-account-provisioning-customizations` | Step Functions hooks during provisioning (e.g., ServiceNow, notify Slack) |

#### L4 — Account request example

```hcl
module "app1_prod" {
  source = "./modules/aft-account-request"

  control_tower_parameters = {
    AccountEmail              = "aws-app1-prod@company.com"
    AccountName               = "app1-prod"
    ManagedOrganizationalUnit = "Prod (ou-xxxx-yyyyyyyy)"
    SSOUserEmail              = "platform-team@company.com"
    SSOUserFirstName          = "Platform"
    SSOUserLastName           = "Team"
  }

  account_tags = {
    owner        = "team-app1"
    cost-center  = "CC1234"
    environment  = "prod"
  }

  change_management_parameters = {
    change_requested_by = "Mridul Singh"
    change_reason       = "New production account for App1"
  }

  custom_fields = {
    vpc_cidr = "10.20.0.0/20"
  }

  account_customizations_name = "prod-baseline"
}
```

#### L5 — Shared networking & security (examples)

```hcl
# Network account: TGW shared to the whole org via RAM
resource "aws_ec2_transit_gateway" "hub" {
  description                     = "org-hub-tgw"
  auto_accept_shared_attachments  = "enable"
  default_route_table_association = "disable"
  default_route_table_propagation = "disable"
}
resource "aws_ram_resource_share" "tgw" {
  name                      = "tgw-share"
  allow_external_principals = false
}
resource "aws_ram_resource_association" "tgw" {
  resource_arn       = aws_ec2_transit_gateway.hub.arn
  resource_share_arn = aws_ram_resource_share.tgw.arn
}
resource "aws_ram_principal_association" "org" {
  principal          = data.aws_organizations_organization.org.arn # or specific OU ARNs
  resource_share_arn = aws_ram_resource_share.tgw.arn
}

# Audit account: GuardDuty auto-enable for ALL new accounts
resource "aws_guardduty_organization_configuration" "this" {
  detector_id                      = aws_guardduty_detector.this.id
  auto_enable_organization_members = "ALL"
}
resource "aws_securityhub_organization_configuration" "this" {
  auto_enable           = false
  auto_enable_standards = "NONE"
  organization_configuration { configuration_type = "CENTRAL" }
}
```

Useful community modules: `aws-ia/control_tower_account_factory/aws`, `terraform-aws-modules/vpc/aws`, `terraform-aws-modules/transit-gateway/aws`, `aws-ia/ipam/aws`.

---

## 6. Azure Landing Zone

### 6.1 Recommended approach

| Option | Notes |
|---|---|
| **ALZ IaC Accelerator → Terraform Azure Verified Modules (AVM) for Platform Landing Zone** ✅ | Microsoft's current recommended Terraform path. Main modules: `Azure/avm-ptn-alz/azurerm` (MG hierarchy, policy, RBAC) and `Azure/avm-ptn-alz-management/azurerm` (Log Analytics, monitoring), plus hub-and-spoke or Virtual WAN connectivity patterns |
| Portal "Azure Landing Zone" accelerator | Quick start / PoC, not ideal for long-term IaC |
| Bicep accelerator | For Bicep-first teams |
| ⚠️ `Azure/caf-enterprise-scale/azurerm` (classic) | **Archived / end-of-life** — do **not** use for new projects; migrate existing deployments to AVM using the official migration tooling (`aka.ms/alz/tf/migrate`) |

### 6.2 Target architecture (recommended management group hierarchy)

```
Tenant Root Group
└── <Org> (intermediate root, e.g. "nitor")      ← most policies assigned here
    ├── Platform
    │   ├── Management     → sub-management   (Log Analytics, Automation, Monitor, Sentinel)
    │   ├── Connectivity   → sub-connectivity (Hub VNet/vWAN, Azure Firewall, VPN/ER GW, Private DNS zones, DDoS)
    │   ├── Identity       → sub-identity     (AD DS DCs / Entra Domain Services if needed)
    │   └── Security       → sub-security     (optional: Sentinel, Defender, Key Vaults)
    ├── Landing Zones
    │   ├── Corp           → internal apps, peered to hub, NO public endpoints
    │   └── Online         → internet-facing apps
    ├── Sandbox            → experimentation, isolated from corp network
    └── Decommissioned     → cancelled subscriptions (deny-all style policies)
```

> Don't assign policies at Tenant Root — use an **intermediate root MG** so you can test/replace the whole hierarchy safely.

### 6.3 Manual (Portal) Setup — Step by Step

**Step 0 — Tenant & identity prep**
1. Entra ID tenant ready; **2 break-glass (emergency) Global Admin accounts** excluded from Conditional Access, with FIDO2 keys + sign-in alerting.
2. Enable **MFA / Conditional Access**, **PIM** (Privileged Identity Management) for all admin roles.
3. Create groups: `alz-platform-owners`, `alz-network-admins`, `alz-security-admins`, `alz-readers`, per-app groups.
4. Global Admin → Entra ID → *Properties* → **"Access management for Azure resources" = Yes** (gives User Access Administrator at `/`). *Turn off after bootstrap.*

**Step 1 — Billing**
- EA: enrollment account owner role; MCA: invoice section **Azure subscription creator** role — needed for automated subscription vending.

**Step 2 — Management groups**
1. Portal → *Management groups* → create the hierarchy above.
2. Hierarchy settings: set **default management group for new subscriptions** to e.g. `Sandbox` (or a dedicated "Onboarding" MG) and require **permission to create MGs**.

**Step 3 — Platform subscriptions**
- Create `sub-management`, `sub-connectivity`, `sub-identity`, (`sub-security`) and move them under their Platform MGs.

**Step 4 — Policy**
1. Assign the **ALZ policy initiatives** (from the ALZ library) at the right scopes, e.g.:
   - `<Org>`: allowed locations, deploy diagnostic settings to Log Analytics, Defender for Cloud plans, Activity Log → Log Analytics, deny classic resources, enforce TLS/HTTPS.
   - `Landing Zones`: deny public IP on NIC, deny RDP/SSH from internet, enforce backup on VMs, deny storage public access, enforce Key Vault soft delete.
   - `Corp`: deny public endpoints, deploy Private DNS zone configs for Private Endpoints.
   - `Sandbox`: deny VNet peering to hub, deny ExpressRoute/VPN.
   - `Decommissioned`: deny all resource creation.
2. For `DeployIfNotExists` / `Modify` policies, the assignment's **managed identity** needs RBAC roles (Contributor / specific roles) at the scope.
3. Run **remediation tasks** for existing resources.

**Step 5 — Management (Logging & Security)**
- In `sub-management`: Log Analytics Workspace, Data Collection Rules, Azure Monitor Agent policies, (Microsoft Sentinel).
- Enable **Defender for Cloud** plans (CSPM, Servers, Storage, Key Vault, Containers, SQL).
- Tenant-level **Activity Log & Entra ID logs** diagnostic settings → workspace.

**Step 6 — Connectivity**
- **Hub & spoke**: hub VNet, Azure Firewall (Premium for TLS inspection), VPN/ExpressRoute Gateway, Azure Bastion, DDoS Protection Plan, **Private DNS zones** for `privatelink.*` + DNS Private Resolver.
- **or Virtual WAN**: vWAN + secured virtual hubs per region.
- UDRs on spokes → Azure Firewall for egress.

**Step 7 — Cost**
- Budgets per subscription, cost-allocation rules, required tags via policy (`Require a tag on resource groups`, `Inherit a tag from the subscription`).

### 6.4 Terraform Setup

#### Bootstrap via ALZ IaC Accelerator (recommended)
The accelerator (PowerShell `ALZ` module) creates: state storage account, managed identities with **federated credentials (OIDC)** for GitHub Actions / Azure DevOps, the repo with starter code, and pipelines (plan/apply with approval).

```powershell
Install-Module -Name ALZ
Deploy-Accelerator -inputs "./inputs.yaml","./platform-landing-zone.tfvars"
```

You pick a **scenario** (e.g., *multi-region hub-and-spoke with Azure Firewall*, *Virtual WAN*, *management only*) and options (regions, DDoS, Bastion, etc.).

#### Core module usage (simplified)

```hcl
terraform {
  required_version = ">= 1.9"
  required_providers {
    azurerm = { source = "hashicorp/azurerm", version = "~> 4.0" }
    alz     = { source = "Azure/alz", version = "~> 0.x" }   # policy/archetype library provider
    azapi   = { source = "Azure/azapi", version = "~> 2.0" }
  }
  backend "azurerm" {
    resource_group_name  = "rg-tfstate"
    storage_account_name = "sttfstatealz001"
    container_name       = "tfstate"
    key                  = "alz-platform.tfstate"
    use_oidc             = true
  }
}

provider "alz" {
  library_references = [{ path = "platform/alz", ref = "<pinned-release>" }]
}

data "azapi_client_config" "current" {}

module "management" {
  source  = "Azure/avm-ptn-alz-management/azurerm"
  version = "<pinned>"
  location                      = "centralindia"
  resource_group_name           = "rg-management-centralindia"
  log_analytics_workspace_name  = "law-platform-centralindia"
  automation_account_name       = "aa-platform-centralindia"
}

module "alz" {
  source  = "Azure/avm-ptn-alz/azurerm"
  version = "<pinned>"

  architecture_name  = "alz"                                   # from library (or your custom architecture)
  parent_resource_id = data.azapi_client_config.current.tenant_id
  location           = "centralindia"

  policy_default_values = {
    log_analytics_workspace_id = jsonencode({ value = module.management.resource_id })
    ama_user_assigned_managed_identity_id = jsonencode({ value = module.management.user_assigned_identity_ids["ama"].id })
  }

  # Override / disable specific assignments per MG
  policy_assignments_to_modify = {
    landingzones = {
      policy_assignments = {
        Deny-IP-forwarding = { enforcement_mode = "Default" }
      }
    }
  }

  subscription_placement = {
    management   = { subscription_id = var.sub_management,   management_group_name = "management" }
    connectivity = { subscription_id = var.sub_connectivity, management_group_name = "connectivity" }
    identity     = { subscription_id = var.sub_identity,     management_group_name = "identity" }
  }
}
```

> **Customising the hierarchy / policies**: create your own *library* (a folder with `archetype_definitions`, `architecture_definitions`, policy JSON) and reference it in the `alz` provider — no need to fork the module.

#### Connectivity (hub & spoke example)
Use the AVM pattern modules for hub networking (e.g., `Azure/avm-ptn-hubnetworking/azurerm`) or Virtual WAN (`Azure/avm-ptn-virtualwan/azurerm`), plus `Azure/avm-ptn-network-private-link-private-dns-zones/azurerm` for Private DNS zones. *(Verify exact names/versions on the Terraform Registry — the accelerator wires these for you.)*

#### Subscription vending (for new application landing zones)

```hcl
module "sub_app1_prod" {
  source  = "Azure/lz-vending/azurerm"   # check registry for the AVM successor name/version
  version = "<pinned>"

  location = "centralindia"

  # 1. Create subscription (alias)
  subscription_alias_enabled = true
  subscription_billing_scope = "/providers/Microsoft.Billing/billingAccounts/<id>/billingProfiles/<id>/invoiceSections/<id>"
  subscription_display_name  = "sub-app1-prod"
  subscription_alias_name    = "sub-app1-prod"
  subscription_workload      = "Production"
  subscription_tags = { owner = "team-app1", environment = "prod", costcenter = "CC1234" }

  # 2. Place in management group → policies inherited immediately
  subscription_management_group_association_enabled = true
  subscription_management_group_id                  = "corp"

  # 3. Spoke VNet peered to hub
  virtual_network_enabled = true
  virtual_networks = {
    spoke = {
      name                    = "vnet-app1-prod"
      address_space           = ["10.40.0.0/22"]
      resource_group_name     = "rg-network-app1-prod"
      hub_peering_enabled     = true
      hub_network_resource_id = var.hub_vnet_id
    }
  }

  # 4. RBAC for the app team
  role_assignment_enabled = true
  role_assignments = {
    app_team_contrib = {
      principal_id   = var.app1_group_object_id
      definition     = "Contributor"
      relative_scope = ""
    }
  }

  # 5. Resource providers to register
  subscription_register_resource_providers_enabled = true
}
```

---

## 7. GCP Landing Zone

### 7.1 Recommended approach

| Option | Notes |
|---|---|
| **Google Cloud Setup** (console guided flow) | Fastest; can **download generated Terraform** at the end — good starting point |
| **Cloud Foundation Fabric – FAST** ✅ *(recommended for enterprises wanting full Terraform control)* | Opinionated, stage-based (bootstrap → resource management → security/networking → project factory), lean modules |
| **terraform-example-foundation** (Google Security Foundations blueprint) | Stages `0-bootstrap`, `1-org`, `2-environments`, `3-networks-*`, `4-projects`, `5-app-infra`; aligned to Google's enterprise foundations blueprint |

### 7.2 Target architecture (recommended folder structure)

```
Organization (company.com)
├── fldr-bootstrap          → prj-b-seed (TF state, SA), prj-b-cicd
├── fldr-common
│   ├── prj-c-logging       → aggregated log sink destination (Log bucket / BigQuery / GCS)
│   ├── prj-c-billing-export→ BigQuery billing export
│   ├── prj-c-scc           → Security Command Center notifications
│   ├── prj-c-kms           → central Cloud KMS keys
│   ├── prj-c-secrets
│   └── prj-net-hub         → hub VPC, Cloud Interconnect / HA VPN, Cloud DNS hub, NCC
├── fldr-network
│   ├── prj-net-prod-host   → Shared VPC host (prod)
│   └── prj-net-nonprod-host→ Shared VPC host (non-prod)
├── fldr-prod               → service projects (app1-prod …)
├── fldr-nonprod            → service projects (app1-dev, app1-qa …)
├── fldr-sandbox            → isolated, budget-capped projects (no Shared VPC)
└── fldr-decommissioned
```

### 7.3 Manual (Console) Setup — Step by Step

**Step 0 — Identity & organization**
1. Verify domain in **Cloud Identity** (Free) or Google Workspace → Organization node is auto-created.
2. Secure **Super Admin** accounts (2+, hardware keys, not used day-to-day).
3. (Recommended) Federate with Entra ID/Okta via **SAML SSO** + user/group **provisioning** (or Google Cloud Directory Sync).
4. Create groups: `gcp-org-admins`, `gcp-billing-admins`, `gcp-network-admins`, `gcp-security-admins`, `gcp-logging-admins`, `gcp-devops`, `gcp-monitoring-admins`.

**Step 1 — Billing**
- Create/link **Cloud Billing account** to the org; grant `roles/billing.admin` to `gcp-billing-admins`, `roles/billing.user` to the Terraform SA (for project creation).
- Enable **billing export to BigQuery** (standard + detailed).

**Step 2 — Google Cloud Setup (guided)**
- Console → **Google Cloud Setup** → walk through: Organization, Users & groups, Administrative access, Billing, Resource hierarchy, Centralized logging, Monitoring, VPC networks, Hybrid connectivity, Security. At the end you can **deploy directly** or **download Terraform**.

**Step 3 — Resource hierarchy**
- IAM & Admin → *Manage resources* → create folders per 7.2. Remove the default `roles/resourcemanager.projectCreator` and `roles/billing.account.creator` granted to **all domain users** at the org.

**Step 4 — Organization Policies (preventive guardrails)** — set at org, override at folder if needed:

| Constraint | Recommended |
|---|---|
| `compute.skipDefaultNetworkCreation` | Enforce |
| `iam.disableServiceAccountKeyCreation` | Enforce |
| `iam.disableServiceAccountKeyUpload` | Enforce |
| `iam.automaticIamGrantsForDefaultServiceAccounts` | Enforce |
| `iam.allowedPolicyMemberDomains` | Your Customer ID only (blocks external/gmail principals) |
| `gcp.resourceLocations` | `in:asia-south1-locations`, `in:asia-south2-locations` |
| `compute.vmExternalIpAccess` | Deny all (allow-list exceptions) |
| `compute.requireOsLogin` | Enforce |
| `compute.requireShieldedVm` | Enforce |
| `storage.uniformBucketLevelAccess` | Enforce |
| `storage.publicAccessPrevention` | Enforce |
| `sql.restrictPublicIp` | Enforce |
| `compute.restrictVpcPeering` / `compute.restrictSharedVpcHostProjects` | Restrict to your org |
| `essentialcontacts.allowedContactDomains` | Your domain |

**Step 5 — Logging**
- Create an **aggregated organization log sink** (`includeChildren = true`) → log bucket in `prj-c-logging` (with retention/lock) and/or BigQuery/GCS.
- Enable **Data Access audit logs** for sensitive services (at org level).

**Step 6 — Security**
- Activate **Security Command Center** (Standard/Premium/Enterprise) at org level; configure notifications → Pub/Sub.
- Optional: **VPC Service Controls** perimeters around sensitive projects (BigQuery, GCS, etc.).
- **Essential Contacts** for security/billing/technical notifications.

**Step 7 — Networking**
- **Shared VPC**: enable host projects (prod & non-prod), create subnets per region, **Cloud NAT**, **Cloud Router**, hierarchical **firewall policies** at org/folder, **Private Google Access / Private Service Connect**.
- Hybrid: **HA VPN** or **Cloud Interconnect**; DNS: Cloud DNS forwarding/peering zones.

### 7.4 Terraform Setup

#### Option A — terraform-example-foundation (stage-based)

| Stage | Creates |
|---|---|
| `0-bootstrap` | Seed project, TF state bucket, Terraform service accounts per stage, CI/CD project (Cloud Build / GitHub Actions / GitLab / Jenkins) |
| `1-org` | Common folders/projects, org policies, log sinks, SCC notifications, KMS, Essential Contacts |
| `2-environments` | Environment folders (`development`, `nonproduction`, `production`) + env-level projects (KMS, secrets) |
| `3-networks-svpc` / `3-networks-hub-and-spoke` | Shared VPCs, firewall policies, DNS, NAT, hybrid connectivity, VPC-SC |
| `4-projects` | Business-unit folders + service projects via project factory |
| `5-app-infra` | Sample workload infra |

#### Option B — Fabric FAST
Stages (names evolve by release; check repo): **org setup / bootstrap** → **resource management (folders + per-team automation)** → **security** & **networking** → **project factory**. Highly YAML-driven factories (org policies, firewall rules, projects defined as YAML).

#### Key snippets (using Google's community modules)

```hcl
terraform {
  required_providers {
    google      = { source = "hashicorp/google",      version = "~> 6.0" }
    google-beta = { source = "hashicorp/google-beta", version = "~> 6.0" }
  }
  backend "gcs" {
    bucket = "bkt-b-tfstate-xxxx"
    prefix = "landing-zone/org"
  }
}

# Folders
resource "google_folder" "prod" {
  display_name = "fldr-prod"
  parent       = "organizations/${var.org_id}"
}
resource "google_folder" "nonprod" {
  display_name = "fldr-nonprod"
  parent       = "organizations/${var.org_id}"
}

# Org policies (v2 API resource)
resource "google_org_policy_policy" "skip_default_network" {
  name   = "organizations/${var.org_id}/policies/compute.skipDefaultNetworkCreation"
  parent = "organizations/${var.org_id}"
  spec { rules { enforce = "TRUE" } }
}

resource "google_org_policy_policy" "resource_locations" {
  name   = "organizations/${var.org_id}/policies/gcp.resourceLocations"
  parent = "organizations/${var.org_id}"
  spec {
    rules {
      values { allowed_values = ["in:asia-south1-locations", "in:asia-south2-locations"] }
    }
  }
}

resource "google_org_policy_policy" "domain_restricted_sharing" {
  name   = "organizations/${var.org_id}/policies/iam.allowedPolicyMemberDomains"
  parent = "organizations/${var.org_id}"
  spec {
    rules { values { allowed_values = [var.customer_id] } } # e.g. C0xxxxxxx
  }
}

# Folder-level override example: sandbox allows external IPs
resource "google_org_policy_policy" "sandbox_external_ip" {
  name   = "${google_folder.sandbox.name}/policies/compute.vmExternalIpAccess"
  parent = google_folder.sandbox.name
  spec { rules { allow_all = "TRUE" } }
}

# Aggregated org log sink → central log bucket
resource "google_logging_organization_sink" "audit" {
  name             = "sk-org-audit-logs"
  org_id           = var.org_id
  include_children = true
  destination      = "logging.googleapis.com/projects/prj-c-logging/locations/asia-south1/buckets/org-audit"
  filter           = "logName: \"/logs/cloudaudit.googleapis.com\""
}
# + grant sink writer identity roles/logging.bucketWriter on the destination

# Hierarchical firewall policy at org
resource "google_compute_firewall_policy" "org" {
  parent     = "organizations/${var.org_id}"
  short_name = "fw-org-baseline"
}
resource "google_compute_firewall_policy_association" "org" {
  firewall_policy   = google_compute_firewall_policy.org.id
  attachment_target = "organizations/${var.org_id}"
  name              = "fw-org-baseline-assoc"
}
```

#### Project Factory (vending new projects)

```hcl
module "app1_prod" {
  source  = "terraform-google-modules/project-factory/google"
  version = "~> 18.0" # pin a tested version

  name              = "prj-p-app1"
  random_project_id = true
  org_id            = var.org_id
  folder_id         = google_folder.prod.id
  billing_account   = var.billing_account

  # Attach as a Shared VPC service project
  svpc_host_project_id = "prj-net-prod-host"
  shared_vpc_subnets = [
    "projects/prj-net-prod-host/regions/asia-south1/subnetworks/sb-p-app1-asia-south1",
  ]

  activate_apis = [
    "compute.googleapis.com", "container.googleapis.com",
    "logging.googleapis.com", "monitoring.googleapis.com",
  ]

  default_service_account = "disable"
  budget_amount           = 2000
  budget_alert_spent_percents = [0.5, 0.8, 1.0]

  labels = { owner = "team-app1", environment = "prod", costcenter = "cc1234" }
}

# App team IAM — grant to a GROUP, not users
resource "google_project_iam_member" "app1_devs" {
  project = module.app1_prod.project_id
  role    = "roles/editor" # prefer narrower predefined/custom roles
  member  = "group:gcp-app1-devs@company.com"
}
```

---

## 8. Adding a New Account / Subscription / Project After Setup

### 8.1 Universal "new account" checklist

**Before creation**
- [ ] Request raised via PR / ticket with: name, owner, cost center, environment, data classification, required network connectivity, CIDR needed
- [ ] Decide **target OU / MG / Folder** (this decides which policies apply!)
- [ ] Allocate **non-overlapping CIDR** from IPAM
- [ ] (AWS) Unique email address created
- [ ] (Azure/GCP) Billing scope / billing account confirmed
- [ ] Check quotas (Organizations account limit, subscription/project quotas)

**During creation (should be automated by vending)**
- [ ] Created via **AFT / subscription vending / project factory** — never manually by clicking (avoids drift)
- [ ] Placed directly into the correct OU/MG/Folder (not left in root / default MG)
- [ ] Mandatory tags/labels applied
- [ ] Baseline applied (logging, security services enrollment, encryption defaults)
- [ ] Network: VPC/VNet/subnets created & attached to hub (TGW attachment / peering / Shared VPC service project)
- [ ] DNS: private zone associations / resolver rules
- [ ] Identity: group-based access assigned (no individual users)
- [ ] Budget + alerts
- [ ] Default VPC/network deleted (AWS: AFT feature flag; GCP: org policy)

**After creation — verify**
- [ ] Account shows **Enrolled / Compliant** (Control Tower) / policy compliance state (Azure) / SCC findings (GCP)
- [ ] Audit logs arriving in the central log account/project
- [ ] Threat detection active (GuardDuty / Defender plans / SCC)
- [ ] Test a **denied action** to confirm guardrails (e.g., create resource in a disallowed region → must fail)
- [ ] Connectivity tested (spoke → hub → on-prem, egress through firewall)
- [ ] Hand over to app team with documentation & access

### 8.2 AWS — new account specifics

| Area | What happens / what to take care of |
|---|---|
| **Creation path** | Use **AFT** (PR to `aft-account-request`) or Control Tower Account Factory. Accounts created directly via `organizations:CreateAccount` are **not** enrolled → no CT baseline; you'd have to enroll them later |
| **CloudTrail** | Org trail covers new account **automatically** |
| **AWS Config** | Control Tower baseline enables Config recorder in governed regions on enrollment |
| **SCPs** | Apply **immediately** based on the OU it lands in |
| **CT controls** | Preventive (SCP-based) apply immediately; detective (Config rules) deployed during enrollment |
| **GuardDuty / Security Hub / Inspector / Macie** | Only automatic if **auto-enable** is configured in the delegated admin. Otherwise manual enrollment needed |
| **IAM Identity Center** | Permission set assignments are **not** automatic for new accounts — add via Terraform (`aws_ssoadmin_account_assignment`) in AFT customizations |
| **Networking** | TGW share via RAM to Org/OU is automatic, but the **VPC + TGW attachment + route table association/propagation** must be done (AFT account customization) |
| **Global customizations** | AFT runs `aft-global-customizations` on every new account (good place for: S3 account-level Block Public Access, EBS default encryption, IAM password policy, budgets, IMDSv2 default) |
| **Moving an existing account in** | Invite → accept → move to OU → **Enroll** in Control Tower (requires `AWSControlTowerExecution` role in the account) |
| **Closing an account** | Move to `Suspended` OU first, then close; it remains in the org ~90 days in `SUSPENDED` state |

### 8.3 Azure — new subscription specifics

| Area | What happens / what to take care of |
|---|---|
| **Creation path** | Subscription vending module (Terraform) using subscription **alias** API with billing scope |
| **Default MG** | A subscription created outside vending lands in the **default management group** (Tenant Root unless you changed it). Set default MG to `Sandbox`/`Onboarding` so stray subscriptions get restrictive policies |
| **Policies** | Inherited **as soon as** the subscription is placed under an MG. Evaluation of new/updated resources is immediate at request time (`Deny` blocks instantly); compliance scan of existing resources runs periodically (~24h) or on-demand |
| **DeployIfNotExists / Modify** | Apply to **new** resources automatically. Pre-existing resources need a **remediation task** |
| **Diagnostic settings / Defender** | Typically enforced via DINE policies at `<Org>` MG → applied automatically |
| **Resource providers** | Must be **registered** in the new subscription (vending can do this) |
| **RBAC** | Role assignments at MG scope are inherited; app team-specific roles must be added at subscription scope |
| **Networking** | Spoke VNet + peering to hub + UDR to firewall (vending module) — **not** automatic. Private DNS zone links handled by DINE policy if using ALZ private DNS policy |
| **Moving subscription between MGs** | Policies of the new MG apply immediately; old ones stop. Existing non-compliant resources are **not deleted**, only reported non-compliant |

### 8.4 GCP — new project specifics

| Area | What happens / what to take care of |
|---|---|
| **Creation path** | Project Factory (Terraform) via the per-stage service account; restrict `projectCreator` to that SA |
| **Org policies** | Inherited from org → folder → project **immediately** on creation |
| **IAM** | Allow policies at org/folder are **inherited and additive** — you cannot remove an inherited grant at the project; use **IAM Deny policies** if you need to block |
| **Log sinks** | Aggregated org/folder sinks with `includeChildren=true` cover the new project automatically |
| **SCC** | Org-level activation covers new projects automatically |
| **APIs** | **Must be enabled** per project (`activate_apis`) |
| **Default network** | Not created if `compute.skipDefaultNetworkCreation` is enforced |
| **Shared VPC** | Must explicitly **attach as service project** and grant `roles/compute.networkUser` on specific subnets |
| **VPC Service Controls** | ⚠️ **NOT inherited** — project must be **explicitly added** to the perimeter (or use dynamic membership via tags in supported setups) |
| **Billing** | Project must be linked to a billing account; add **budget** |
| **Moving a project between folders** | New folder's org policies + IAM apply immediately; old folder's inherited IAM is removed. Requires `resourcemanager.projects.move` on both source and destination |

---

## 9. How Policies Affect a New Account (Inheritance Deep-Dive)

### 9.1 Comparison

| Behavior | AWS (SCP / RCP) | Azure Policy | GCP Org Policy | GCP IAM |
|---|---|---|---|---|
| Inherited down the tree? | ✅ Yes | ✅ Yes | ✅ Yes (unless overridden) | ✅ Yes |
| How multiple levels combine | **Intersection** — action must be allowed by SCPs at *every* level (root → OU → account) | **Union of all assignments** — every assignment applies; a `Deny` anywhere wins | Child can **inherit**, **merge** (list constraints), or **replace/reset** parent policy | **Additive** (union). Use IAM Deny to subtract |
| Can child loosen parent? | ❌ No (an Allow at child can't exceed parent) | ❌ No (only via **exemptions** / `notScopes` on the assignment) | ✅ Yes, if child sets its own policy (needs `orgpolicy.policyAdmin`) | ❌ No (only Deny policies restrict) |
| Applies to top/admin account? | ❌ SCPs don't affect the **management account** | ✅ Applies to all subs under the MG | ✅ | ✅ |
| Grants permissions? | ❌ Never — only sets the maximum | ❌ (governs resources, not identities) | ❌ | ✅ |
| Effect on existing resources | Not retroactive; blocks future API calls | `Deny` not retroactive (reports non-compliant); DINE/Modify need remediation | Mostly not retroactive (e.g., existing external IPs remain) | Immediate |
| Limits to know | Max **5 SCPs** attached per target; **5,120 chars** per SCP; OU depth max 5 | Initiatives, assignment & exemption limits per scope; MG depth max 6 (excl. root & sub level) | Folder depth max 10 | — |

### 9.2 Worked examples

**AWS** — Root has `FullAWSAccess` + `deny-regions (allow ap-south-1, ap-south-2)`; `Workloads/Prod` has `deny-public-s3`; a new account `app1-prod` is placed in `Prod`:
- Effective = FullAWSAccess ∩ deny-regions ∩ deny-public-s3.
- Developer with `AdministratorAccess` in `app1-prod` **cannot** create EC2 in `us-east-1` and **cannot** make S3 public.
- If later moved to `Sandbox` → `deny-public-s3` no longer applies instantly.
- ⚠️ If you remove `FullAWSAccess` from an OU without adding an allow-list SCP, **everything is denied** in all its accounts.

**Azure** — `<Org>` MG has *Allowed locations = centralindia, southindia*; `Corp` MG has *Deny public IP*. New subscription placed under `Corp`:
- Both apply. Creating a VM in `eastus` → denied. Creating a public IP → denied.
- An exemption for a specific resource group can waive *Deny public IP* (e.g., for an App Gateway), with expiry date and justification.

**GCP** — Org enforces `compute.vmExternalIpAccess = deny all`; `fldr-sandbox` sets `allow_all`. New project in `fldr-sandbox`:
- External IPs allowed (child **replaced** parent). Same project moved to `fldr-prod` → external IPs denied for **new** VMs.
- `gcp-org-admins` granted `roles/viewer` at org → automatically viewers on new project (cannot be removed at project level).

### 9.3 Safe policy rollout process (all clouds)

1. Write policy as code → PR → peer review.
2. Deploy in **Audit / dry-run mode** first (Azure `enforcementMode = DoNotEnforce`, GCP **dry-run org policies**, AWS → attach to `Policy-Staging` OU with a test account).
3. Review impact (compliance reports, CloudTrail `AccessDenied` events, Policy Simulator in GCP).
4. Communicate to app teams with date.
5. Enforce at a **non-prod** scope → then **prod**.
6. Maintain an **exception process** (Azure exemptions with expiry; GCP folder-level overrides; AWS condition keys on approved role ARNs).

---

## 10. Recommended Terraform Repository Layout

```
landing-zone/
├── README.md
├── docs/                      # ADRs, IP plan, naming, RACI
├── modules/                   # thin internal wrappers around community/AVM modules
│   ├── aws-account-baseline/
│   ├── azure-sub-baseline/
│   └── gcp-project-baseline/
├── aws/
│   ├── 00-bootstrap/          # state bucket, OIDC provider & roles
│   ├── 01-organization/       # OUs, SCPs, RCPs, tag/backup policies, CT controls
│   ├── 02-aft/                # AFT module deployment
│   ├── 03-security/           # delegated admin, GuardDuty/SecHub org config (Audit acct)
│   ├── 04-network/            # IPAM, TGW, egress/inspection, R53 resolver (Network acct)
│   └── aft-repos/             # (or separate repos) account-request, global/account customizations
├── azure/
│   ├── 00-bootstrap/          # generated by ALZ accelerator (state, identities, pipelines)
│   ├── 01-platform/           # avm-ptn-alz + management + connectivity
│   ├── lib/                   # custom ALZ library: archetypes, policies
│   └── 02-subscriptions/      # one file / tfvars per vended subscription
├── gcp/
│   ├── 0-bootstrap/
│   ├── 1-org/
│   ├── 2-environments/
│   ├── 3-networks/
│   └── 4-projects/            # project factory inputs (YAML/tfvars per project)
├── policies/                  # policy-as-code tests (OPA/Conftest, Checkov, tfsec/Trivy)
└── .github/workflows/         # plan on PR, apply on merge with approvals, per cloud/stage
```

**Principles**
- Separate **state per stage / per cloud / per environment** (small blast radius).
- **Pin** provider and module versions; upgrade via PR with plan review.
- Each stage gets its **own CI identity** with least privilege (e.g., the GCP networking SA can't modify org policies).
- Use `tfvars`/YAML inputs for vending so adding an account = adding a file.

---

## 11. CI/CD, State & Security for the Landing Zone Code

| Topic | AWS | Azure | GCP |
|---|---|---|---|
| State backend | S3 (versioning, KMS, Object Lock optional) + S3 lockfile / DynamoDB | Storage Account (blob versioning, soft delete, private endpoint) | GCS bucket (versioning, CMEK, uniform access) |
| CI auth | **GitHub OIDC → IAM role** | **Federated credentials on Managed Identity / App Registration** | **Workload Identity Federation → service account** |
| Never | Long-lived access keys in CI | Client secrets in CI variables | SA JSON keys |

**Pipeline stages:** `fmt` → `validate` → `tflint` → security scan (Checkov / Trivy) → policy check (OPA/Conftest) → `plan` (posted to PR) → manual approval → `apply`. Run a scheduled **drift detection** `plan` (e.g., nightly) and alert on changes.

---

## 12. IP Address Planning (Multi-Cloud)

Reserve large, non-overlapping supernets per cloud & region, then carve per environment:

| Cloud / Region | Supernet | Example split |
|---|---|---|
| On-prem | 10.0.0.0/12 | existing |
| AWS ap-south-1 | 10.16.0.0/12 | prod 10.16.0.0/13, non-prod 10.24.0.0/13 |
| AWS ap-south-2 (DR) | 10.32.0.0/13 | |
| Azure centralindia | 10.40.0.0/13 | hub 10.40.0.0/22, spokes from 10.40.4.0+ |
| Azure southindia (DR) | 10.48.0.0/13 | |
| GCP asia-south1 | 10.56.0.0/13 | prod host VPC / non-prod host VPC subnets |
| GCP asia-south2 (DR) | 10.64.0.0/13 | |
| GKE/AKS/EKS pod ranges | 100.64.0.0/10 (CGNAT) or secondary ranges | avoid exhausting RFC1918 |

Manage it centrally (AWS VPC IPAM, Azure Virtual Network Manager IPAM, or an external IPAM / NetBox) and feed allocations into vending inputs.

---

## 13. Common Pitfalls

1. **Workloads in the AWS management account** — SCPs don't protect it.
2. **Choosing the wrong Control Tower home region** — can't change it later.
3. **Creating accounts outside AFT** → unenrolled, no baseline, drift.
4. **Forgetting auto-enable** in GuardDuty/Security Hub → new accounts unmonitored.
5. **Assigning Azure policies at Tenant Root** instead of an intermediate MG.
6. **Leaving Azure default MG as Tenant Root** → new subscriptions land ungoverned.
7. **DINE policies without RBAC on the managed identity** → remediation silently fails.
8. **Using the archived CAF enterprise-scale Terraform module** for new Azure builds.
9. **GCP: not removing default `projectCreator` / `billing.account.creator` from all domain users.**
10. **GCP: assuming VPC Service Controls inherit** — they don't.
11. **Overlapping CIDRs** between clouds/on-prem → painful re-IP later.
12. **Mixing console changes with Terraform** → drift; lock down console write access for platform resources.
13. **Hitting SCP limits** (5 per target / 5,120 chars) → design consolidated SCPs early.
14. **No break-glass process** → locked out after a bad policy.
15. **Huge single Terraform state** for the whole org → slow plans, big blast radius.

---

## 14. Day-2 Operations

- **Upgrades**: Control Tower landing zone version updates (then re-register OUs / update accounts), AFT releases, ALZ library/AVM module releases, FAST/foundation releases.
- **Drift**: Control Tower drift notifications, nightly `terraform plan`, Azure Policy compliance, SCC findings.
- **Access reviews**: quarterly review of admin groups, PIM activations, break-glass usage.
- **Cost reviews**: monthly per-account/subscription/project review, rightsizing, commitments (Savings Plans / Reservations / CUDs).
- **Policy lifecycle**: new requirement → staging → audit → enforce → review exceptions on expiry.
- **Decommissioning**: move to `Suspended`/`Decommissioned` → export required data/logs → close account/cancel subscription/shut down project (GCP projects are recoverable for 30 days).
- **Documentation**: keep this README, ADRs, IP plan and onboarding guide for app teams up to date.

---

## 15. References

**AWS**
- AWS Control Tower User Guide — https://docs.aws.amazon.com/controltower/latest/userguide/
- Account Factory for Terraform (AFT) — https://docs.aws.amazon.com/controltower/latest/userguide/aft-overview.html
- AFT Terraform module — https://github.com/aws-ia/terraform-aws-control_tower_account_factory
- Organizing Your AWS Environment Using Multiple Accounts (whitepaper)
- Landing Zone Accelerator on AWS — https://aws.amazon.com/solutions/implementations/landing-zone-accelerator-on-aws/

**Azure**
- Azure Landing Zones (CAF) — https://learn.microsoft.com/azure/cloud-adoption-framework/ready/landing-zone/
- ALZ Terraform (AVM) docs — https://aka.ms/alz/tf
- ALZ IaC Accelerator — https://aka.ms/alz/acc
- Migration from caf-enterprise-scale — https://aka.ms/alz/tf/migrate
- Azure Verified Modules — https://aka.ms/avm

**GCP**
- Enterprise foundations blueprint — https://cloud.google.com/architecture/security-foundations
- terraform-example-foundation — https://github.com/terraform-google-modules/terraform-example-foundation
- Cloud Foundation Fabric / FAST — https://github.com/GoogleCloudPlatform/cloud-foundation-fabric
- Project Factory module — https://github.com/terraform-google-modules/terraform-google-project-factory
- Landing zone design in Google Cloud — https://cloud.google.com/architecture/landing-zones

---

> ⚠️ **Version note:** Module names, versions, and Control Tower landing zone versions change frequently. Always pin and verify against the Terraform Registry and official docs before applying. Code snippets here are **reference templates** — review, test in a sandbox organization/tenant, and adapt before production use.
