# Hybrid Cloud Migration — On-Premises to Azure

End-to-end hybrid cloud migration project: migrating Pop's Paper Company from an on-premises Windows Server 2022 environment to Microsoft Azure, covering identity synchronization, file storage migration, network security, and backup.

---

## Architecture Overview

![Architecture Diagram](architecture/architectural-diagram.png) 

On-premises Active Directory provides identity, synchronized to Microsoft Entra ID via Entra Connect. Azure provides scalable file storage (Azure Files), backup (Recovery Services Vault), and secrets management (Key Vault). The Storage Account uses a public endpoint secured by the built-in Storage Firewall, avoiding the cost of a Private Endpoint and VPN Gateway.

---

## Technologies Used

**On-Premises**
- Windows Server 2022 (VMware)
- Active Directory Domain Services
- DNS, DHCP, File Server
- Active Directory Certificate Services (Internal CA, LDAPS)
- Windows Server Backup
- Group Managed Service Account (gMSA)

**Azure**
- Resource Groups, Virtual Network, Subnets, NSG
- Storage Account, Azure Files
- Recovery Services Vault
- Key Vault
- Microsoft Entra ID, Entra Connect
- Azure Migrate, AzCopy

**Tools**
- Azure Portal
- VMware Workstation
- Azure Pricing Calculator, TCO Calculator

---

## Part 1 — On-Premises Setup (OSL-DC-1)

### Active Directory Domain Services

Windows Server 2022 promoted to Domain Controller for `popspaper.com`. Forest created with DNS, DHCP, and File Server roles installed.



![AD Installation](screenshots/01-onprem-setup/01-AD-installation.png)




![DC Promotion](screenshots/01-onprem-setup/02-DC-promotion.png)




![File Storage Role Installation](screenshots/01-onprem-setup/03-file-storage-role-installation.png)


### Users and Security Groups

8 user accounts created across 6 departments, following the AGDLP model — users in Global Groups, Global Groups nested in Domain Local Groups, permissions assigned to Domain Local Groups.

| User | Department | Role |
|---|---|---|
| Alice Morgan | HR | HR Coordinator |
| Jung Park | Finance | Account Officer |
| Robin Hayes | Sales | Sales Executive |
| Alexis Turner | Production | Production Head |
| Mira Patel | IT & Security | Software Developer |
| Vivek Anand | IT & Security | Jr. Analyst |
| Ruby Rox | Customer Support | Support Representative |
| Rune Solberg | Executive | CTO |


![Domain Users Created](screenshots/01-onprem-setup/04-domain-users-created.png)


![Domain Joined Computers](screenshots/01-onprem-setup/05-domain-joined-computers.png)


![Domain Local Groups](screenshots/01-onprem-setup/06-domain-local-groups.png)


![Global Groups](screenshots/01-onprem-setup/07-global-groups.png)


### File Server and NTFS Permissions

Department folders created with NTFS permissions applied via Domain Local Groups, mirroring department access boundaries.


![NTFS Permissions](screenshots/01-onprem-setup/08-NTFS-permissions.png)


![Share Permissions](screenshots/01-onprem-setup/09-share-permissions.png)


---

## Part 2 — Certificate Authority, LDAPS and Backup

### Certificate Authority and LDAPS

Internal Root CA installed on OSL-DC-1. LDAPS certificate issued, enabling encrypted LDAP authentication on port 636. Computer certificates auto-enrolled to domain-joined machines via Group Policy.


![CA Issued Certificates](screenshots/02-onprem-ca-backup/01-CA-issued-certificates.png)


![LDAPS Certificate](screenshots/02-onprem-ca-backup/02-LDAPS-certificate.png)


### Firewall Configuration

| Service | Port | Protocol |
|---|---|---|
| DNS | 53 | TCP/UDP |
| Kerberos | 88 | TCP/UDP |
| LDAP | 389 | TCP/UDP |
| SMB | 445 | TCP |
| RPC Endpoint Mapper | 135 | TCP |
| Dynamic RPC | 49152–65535 | TCP |
| LDAPS | 636 | TCP |
| ICMP | N/A | ICMPv4 |

### Backup Server

Dedicated backup server with 100GB SSD. Daily backup schedule covering Active Directory, DNS, Certificate Authority, Group Policy, and shared folders.


![On-Premises Backup Schedule](screenshots/02-onprem-ca-backup/03-Onprem-backup-schedule.png)


---

## Part 3 — Project Planning

### Migration Critical Path

Migration planned using a critical path covering analysis, planning, Azure preparation, identity sync, storage migration, testing, and cutover.


![Migration Critical Path](screenshots/03-project-planning/01-Migration-critical-path.png)


---

## Part 4 - Azure Migrate Discovery

### Migration Estimate using Azure Migrate

Azure Migrate appliance deployed to discover and inventory on-premises servers, file shares, and services before migration.


![Azure Migrate Setup](screenshots/04-azure-migrate/01-Azure-migrate-setup.png)


![Azure Migrate Scanning](screenshots/04-azure-migrate/02-Azure-migrate-scanning.png)


![Azure Migrate Inventory](screenshots/04-azure-migrate/03-Azure-migrate-inventory.png)


![Azure Migrate Discovered Resources](screenshots/04-azure-migrate/04-Azure-migrate-discovered-resources.png)


---

## Part 5 - Cost Analysis

### Azure Pricing Calculator

Estimated monthly Azure cost: **$168.10**


![Azure Pricing Calculator](screenshots/05-cost-analysis/01-Azure-pricing-calculator.png)


### TCO Comparison - On-Premises vs Azure

| | On-Premises (1 year) | Azure (1 year) |
|---|---|---|
| Total Cost | $299,530 | $34,471 |
| Compute | $3,458.64 | $2,647.44 |
| Storage | $229,802.00 | $31,627.87 |
| Data Center | $64,993.38 | $0.00 |
| Networking | $1,017.96 | $1.20 |
| IT Labor | $258.50 | $194.00 |

Migrating storage to Azure reduces total cost of ownership by approximately 88% over one year, primarily by eliminating data center and hardware costs.


![TCO On-Premises vs Azure](screenshots/05-cost-analysis/02-TCO-onprem-vs-azure.png)


---

## Part 6 - Azure Infrastructure

### Resource Group

All resources deployed in `rg-hybrid-prod-noeast-001`, Norway East.

### Storage Account and File Share

Storage account `hybridprodnoeast002` with file share `fs-hybrid-prod-noeast-001`.


![Storage Account Overview](screenshots/06-azure-setup/01-storage-account-overview.png)


![File Share Overview](screenshots/06-azure-setup/02-file-share-overview.png)


### Virtual Network and Subnets

VNet `vnet-hybrid-prod-noeast` with two subnets:
- `default` - 10.0.0.0/24
- `vnet-pe-files` - 10.0.1.0/24


![VNet Creation](screenshots/06-azure-setup/03-vnet-creation.png)


![Subnet Configuration](screenshots/06-azure-setup/04-subnet-configuration.png)


### Public Endpoint and SAS Token

Public endpoint enabled on Storage Account. SAS token generated for AzCopy migration.


![Public Endpoint Enabled](screenshots/06-azure-setup/05-public-endpoint-enabled.png)


![SAS Token Generation](screenshots/06-azure-setup/06-sas-token-generation.png)


### Storage Account Firewall

The Storage Account public endpoint was initially set to "Enabled from all networks." This was hardened to "Enabled from selected virtual networks and IP addresses," whitelisting the on-premises network's public IP — restricting network-layer access while maintaining the cost-free public endpoint approach.

---

## Part 7 - Entra Connect Sync

### Entra Connect Service Account - gMSA

Microsoft Entra Connect automatically provisioned a Group Managed Service Account (gMSA) during installation:

- **Account name:** provAgentgMSA
- **Type:** msDS-groupManagedServiceAccount
- **Purpose:** Azure AD cloud sync service account

The gMSA runs the Entra Connect sync service on OSL-DC-1 with automatic password rotation managed by the Domain Controller. This eliminates the risk of a static service account password on a privileged account that has AD read/write access for identity synchronisation.

### AD and Entra ID Synchronization

Microsoft Entra Connect installed on OSL-DC-1. OU filtering applied - only the PopsPaper OU synced, excluding built-in containers. 8 users, Domain Local Security Groups, and Global Security Groups synced to Entra ID.


![Entra Connect OU Filtering](screenshots/07-entra-connect/01-entra-connect-ou-filtering.png)


![Entra ID Synced Users](screenshots/07-entra-connect/02-entra-id-synced-users.png)


---

## Part 8 - File Migration

### AzCopy Migration (Proof of Concept)

AzCopy used with a SAS token to migrate the IT & Security department folder from on-premises to Azure File Share, demonstrating the migration process end-to-end.

The same process applies to the remaining five department folders (HR, Finance, Sales, Production, Customer Support) - migration was scoped to one folder due to time constraints.


![AzCopy Migration Running](screenshots/08-file-migration/01-azcopy-migration-running.png)


![File Share Migrated Folders](screenshots/08-file-migration/02-file-share-migrated-folders.png)


![File Share On-Premises Access](screenshots/08-file-migration/03-file-share-onprem-access.png)


![File Share Mapped Drive](screenshots/08-file-migration/04-file-share-mapped-drive.png)


---

## Part 9 - RBAC and Identity Access

### Identity-Based Access and RBAC

Identity-based access enabled on the File Share via Azure Portal, connected to synced Entra ID identities. Role `Storage File Data SMB Share Elevated Contributor` assigned to Vivek Anand at the File Share level.


![Identity Based Access Enabled](screenshots/09-rbac-identity/01-identity-based-access-enabled.png)


![RBAC Role Assignment](screenshots/09-rbac-identity/02-rbac-role-assignment.png)


---

## Part 10 - Network Security Group

### NSG Configuration

NSG `nsg-hybrid-prod-noeast001` created and attached to the `vnet-pe-files` subnet.

| Priority | Rule | Action |
|---|---|---|
| 100 | Allow VNet inbound | Allow |
| 65000 | Allow Azure Load Balancer | Allow |
| 65500 | Deny all inbound | Deny |


![NSG Creation](screenshots/10-nsg/01-nsg-creation.png)


![NSG Inbound Rules](screenshots/10-nsg/02-nsg-inbound-rules.png)


![NSG Subnet Association](screenshots/10-nsg/03-nsg-subnet-association.png)


---

## Part 11 - Azure Backup and Key Vault

### Recovery Services Vault

`rsv-hybrid-prod-noeast` configured with daily backup policy (`DailyBackup-01`), retention 30 days.


![Recovery Services Vault](screenshots/11-backup-keyvault/01-recovery-services-vault.png)


![Backup Policy Created](screenshots/11-backup-keyvault/02-backup-policy-created.png)


![Backup Enabled File Share](screenshots/11-backup-keyvault/03-backup-enabled-fileshare.png)


### Key Vault

`hybridprod-noeast` provisioned with Azure RBAC permission model, Norway East region.

The vault was configured and access control established during this project. No secret or certifications were stored. In continouation vault would store:

1. LDAPS certification imported from internal CA (OSL-DC-1)
2. Storage account access Key
3. Future application secrets and service principle credentials.  


![Key Vault Creation](screenshots/11-backup-keyvault/04-key-vault-creation.png)

---

## Architecture Decision - Public Endpoint vs Private Endpoint

A Private Endpoint was considered but not implemented:

**Cost:** A Private Endpoint places the Storage Account on a private IP inside the VNet, unreachable from on-premises over the public internet. Bridging this requires an Azure VPN Gateway (~$140+/month) and a Private DNS Resolver (~$115+/month) - nearly tripling the project's estimated monthly cost of $168.10.

**On-premises infrastructure:** A Site-to-Site VPN also requires compatible VPN hardware at the on-premises location, outside the scope of this VMware-based lab.

**Decision:** A Public Endpoint secured by the Storage Account Firewall (IP whitelisting) provides network-layer access control at zero additional cost - aligning with PopsPaper's limited budget while preventing unauthorized access from arbitrary internet locations.

In a production environment with a larger budget or existing Site-to-Site VPN, a Private Endpoint would be the preferred approach for full network isolation.

---

## Testing and Verification

| Test | Method | Result |
|---|---|---|
| Users synced to Entra ID | Entra ID → Users | ✅ Pass - 8 users present |
| Groups synced to Entra ID | Entra ID → Groups | ✅ Pass - Domain Local + Global groups present |
| gMSA provisioned for Entra Connect | AD → Managed Service Accounts | ✅ Pass - provAgentgMSA present |
| File migrated to Azure | File Share → Browse | ✅ Pass - IT & Security folder present |
| File accessible from on-prem | Mapped Z: drive | ✅ Pass - folder visible and accessible |
| RBAC access works | Login as Vivek, access IT & Security folder | ✅ Pass |
| NSG rules applied | NSG → Inbound rules | ✅ Pass - Deny all inbound at priority 65500 |
| Backup policy active | Recovery Services Vault → Backup Items | ✅ Pass - DailyBackup-01 active |
| Storage Firewall restricts access | Storage Account → Networking | ✅ Pass - only whitelisted IP allowed |

---

## Troubleshooting

### GPO Filtering

Some Group Policy settings intended for the IT & Security department were not applying correctly to client machines. Resolved by reviewing and correcting GPO filtering and OU linking.

### Firewall Blocking SMB/RPC

AD synchronization initially failed due to Windows Defender Firewall blocking SMB and RPC communication between domain controllers. Resolved by reviewing firewall rules and enabling the required ports listed in the firewall configuration table above.

---

## AI and External Tools

**AI assistance:** Claude (Anthropic) was used during the
post-submission improvement phase of this project to:

- Review and correct architectural inaccuracies
- Improve README structure and documentation quality
- Generate the architecture diagram prompt and review iterations based on the original exam project submission

**All technical implementation, Azure Portal configuration, VMware setup, and Active Directory work was performed manually without AI assistance.**

**External tools used:**

- Azure Pricing Calculator — cost estimation
- Azure TCO Calculator — on-premises vs cloud cost comparison
- Azure Migrate — on-premises discovery and inventory
- AzCopy — file migration
- draw.io / external diagram tool — architecture diagram
- VMware Workstation — on-premises virtual environment

---

## What I Learned

- Hybrid identity synchronization using Microsoft Entra Connect, including OU filtering and security group sync
- Group Managed Service Accounts (gMSA) - automatic password rotation for privileged service accounts running on domain-joined servers
- Cloud migration planning using Azure Migrate for discovery and inventory
- File migration to Azure Storage using AzCopy and SAS tokens
- Network security design - VNet, subnets, and NSG rule configuration
- RBAC role assignment for granular Azure resource access
- Cost-driven architecture decisions - evaluating Public Endpoint + Storage Firewall against Private Endpoint + VPN Gateway based on budget constraints
- Internal Certificate Authority deployment and LDAPS configuration for encrypted directory authentication

---

## Future Improvements

- Migrate remaining five department folders (currently one migrated as proof of concept)
- Implement per-folder NTFS permission enforcement across all departments once fully migrated
- Evaluate Entra ID Kerberos for identity-based file share authentication as a modern alternative
- Store certificates and secrets in Key Vault (currently provisioned but empty)
- Consider Private Endpoint + VPN Gateway if budget allows, for full network isolation
- Configure Storage Account Firewall to whitelist on-premises public IP for additional network-layer security
