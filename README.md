# NTFS File Server Lab
 
Azure infrastructure-as-code for Windows Server administration — Active Directory, NTFS access control, SMB file services, and Group Policy. Deployed with Terraform, configured automatically with PowerShell and Azure Run Command.
 
---
 
## What You'll Learn
 
- Deploying Windows Server infrastructure on Azure using Terraform
- Storing secrets securely with Azure Key Vault (RBAC-based)
- Managing Terraform state remotely in Azure Blob Storage
- Promoting a server to an Active Directory Domain Controller
- Automating VM configuration without RDP using `az vm run-command`
- Configuring NTFS permissions and SMB file shares
- Implementing group-based access control
- Managing Group Policy Objects (GPO) for RDP access
- Joining Windows clients to a domain
---
 
## What This Lab Simulates 
 
Picture a small company with four departments: Finance, HR, Sales, and IT. Each department has its own shared folder on a central file server — like a locked filing cabinet that only the right employees can open. IT can open every cabinet because they manage the whole office.
 
This lab builds that setup automatically in the cloud:
 
- A **Domain Controller** — think of this as the security desk at a corporate office. Every user, computer, and permission flows through here. No one gets access to anything without checking in first.
- A **File Server** — the actual cabinet holding department folders, each locked down so only the right group can get in.
- A **Client Workstation** — a regular Windows PC you log into as a test employee to experience the access rules firsthand.
- **Group Policy** — a rulebook that gets pushed automatically to every PC in the company. Write the rule once on the Domain Controller, and it applies everywhere.
---
 
## Architecture
 
```
┌─────────────────────────────────────────────────────────────┐
│                        Azure VNet                           │
│                        10.0.0.0/16                          │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                  Subnet: 10.0.1.0/24                  │  │
│  │                                                       │  │
│  │   ┌─────────┐    ┌─────────┐    ┌─────────────┐      │  │
│  │   │  DC01   │    │  FS01   │    │  CLIENT01   │      │  │
│  │   │ (AD DS) │◄───│ (Files) │◄───│ (Workstation│      │  │
│  │   └─────────┘    └─────────┘    └─────────────┘      │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                             │
│   ┌─────────────────────┐   ┌──────────────────────────┐   │
│   │   Azure Key Vault   │   │  Azure Storage Account   │   │
│   │  (VM credentials)   │   │   (Terraform state)      │   │
│   └─────────────────────┘   └──────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```
 
| VM | Role | OS |
|----|------|----|
| DC01 | Domain Controller | Windows Server 2022 |
| FS01 | File Server | Windows Server 2022 |
| CLIENT01 | Client Workstation | Windows 11 Pro |
 
- **Domain:** lab.local
- **Region:** Central US
- **VM Size:** Standard_B2as_v2
> 💰 **Cost Estimate:** Running all three VMs costs approximately $0.15–0.25/hour. Run `az group delete -n RG-FileServerLab --yes` when finished to avoid unexpected charges.
 
---
 
## Prerequisites
 
| Tool | Version | Install |
|------|---------|---------|
| Terraform | >= 1.5.0 | [Download](https://developer.hashicorp.com/terraform/downloads) |
| Azure CLI | Latest | `winget install Microsoft.AzureCLI` |
| Active Azure subscription | — | [portal.azure.com](https://portal.azure.com) |
 
---
 
## Pre-Setup — Remote State Backend (One Time)
 
Terraform keeps a record of everything it builds called a **state file** — think of it as Terraform's memory. By default that file sits on your local machine, which is risky (lose your laptop, lose your state). This step moves it to Azure Blob Storage so it is encrypted at rest, never sits on local disk, and can be shared across machines.
 
```bash
# Create a dedicated resource group for state storage
az group create --name RG-TerraformState --location "Central US"
 
# Create a storage account (name must be globally unique, 3-24 lowercase chars)
az storage account create --name <YOUR_STORAGE_ACCOUNT_NAME> \
    --resource-group RG-TerraformState \
    --sku Standard_LRS \
    --encryption-services blob
 
# Create the container
az storage container create --name tfstate \
    --account-name <YOUR_STORAGE_ACCOUNT_NAME>
```
 
Then open `backend.tf` and replace `REPLACE_WITH_YOUR_STORAGE_ACCOUNT_NAME` with the name you chose above.
 
> **Note:** This step only needs to be done once. If you are continuing from a previous lab and already have a `RG-TerraformState` storage account, skip this and just update `backend.tf` with that account name.
 
---
 
## Step 1 — Configure Variables
 
```bash
cp terraform.tfvars.example terraform.tfvars
```
 
Open `terraform.tfvars` and set your values:
 
```hcl
location       = "Central US"
rdp_source     = "YOUR_PUBLIC_IP/32"   # Find yours at whatismyip.com
server_vm_size = "Standard_B2as_v2"
client_vm_size = "Standard_B2as_v2"
```
 
Set the admin password as an environment variable — it never touches disk:
 
```powershell
# PowerShell
$env:TF_VAR_admin_password = "YourStrongPassword!"
```
```bash
# Bash / Azure Cloud Shell
export TF_VAR_admin_password="YourStrongPassword!"
```
 
> **Why an environment variable instead of just typing it in?** Terraform automatically reads any `TF_VAR_*` environment variable as the matching input variable. The password exists in memory only — it is never written to a file, never appears in `terraform.tfvars`, and can never accidentally get committed to GitHub. This is the enterprise-standard approach.
 
> **Password requirements:** Min 12 characters, must include uppercase, lowercase, a number, and a symbol.
 
> **Security note:** `terraform.tfvars` is excluded from git via `.gitignore`. The password is also stored in Azure Key Vault during deployment, so you never need to type it again after `terraform apply`.
 
---
 
## Step 2 — Deploy Infrastructure
 
```bash
# Login to Azure
az login
 
# Initialise providers and connect to remote state backend
terraform init
 
# Preview what will be created
terraform plan
 
# Deploy (approx. 10–15 minutes)
terraform apply
```
 
After `terraform apply` completes, note the Key Vault name from the output:
 
```
Outputs:
 
key_vault_name     = "kv-fslab-a1b2c3d4"   ← copy this
dc01_public_ip     = "20.x.x.x"
fs01_public_ip     = "20.x.x.x"
client01_public_ip = "20.x.x.x"
```
 
> **What just got deployed:** 3 VMs (DC01, FS01, CLIENT01), a virtual network, firewall rules (NSG), public IPs, and an Azure Key Vault that holds the admin password. From this point forward you never need to type the password manually — the automation pulls it from Key Vault automatically.
 
---
 
## Step 3 — Configure the Lab (Automated)
 
This single command does everything that would normally require hours of manual work inside RDP sessions. It pushes PowerShell scripts directly to each VM through the Azure agent using `az vm run-command` — no WinRM, no extra firewall rules needed.
 
```powershell
.\configure-lab.ps1 -KeyVaultName "kv-fslab-a1b2c3d4"
```
 
Here is what runs automatically, in order:
 
| Stage | What Happens (Plain English) | VM |
|-------|-----------------------------|----|
| 1 | Turns DC01 into a Domain Controller — makes it the "security desk" for lab.local | DC01 |
| 2 | Creates the department folders (OUs), employee groups, and test user accounts in Active Directory | DC01 |
| 3 | Introduces FS01 to the domain — DC01 now recognises it as a trusted member of the network | FS01 |
| 4 | Creates the shared department folders and locks each one down with NTFS permissions | FS01 |
| 5 | Introduces CLIENT01 to the domain — the test workstation is now managed by DC01 | CLIENT01 |
| 5b | Grants domain users permission to Remote Desktop into CLIENT01 | CLIENT01 |
| 6 | Writes and pushes a Group Policy rule for RDP access to all domain computers | DC01 |
| 7 | Runs automated checks — prints a PASS/FAIL report for every AD object and share permission | DC01 + FS01 |
 
**Total time: approximately 15–20 minutes, fully unattended.**
 
You will see live output as each stage completes. A `[PASS]` / `[FAIL]` report prints at the end confirming everything is configured correctly.
 
---
 
## Step 4 — Verify the Lab
 
RDP into CLIENT01 using the public IP from `terraform output`:
 
```
Username : LAB\sarah.jones
Password : P@ssw0rd123!
```
 
Open File Explorer and navigate to `\\FS01`. Test the access scenarios below:
 
| User | Share | Expected Result |
|------|-------|----------------|
| sarah.jones (GRP_Finance) | `\\FS01\Finance` | ✅ Can read and write |
| sarah.jones (GRP_Finance) | `\\FS01\HR` | ❌ Access Denied |
| lisa.white (GRP_HR) | `\\FS01\Finance` | 👁️ Read only |
| lisa.white (GRP_HR) | `\\FS01\HR` | ✅ Can read and write |
| john.smith (GRP_IT) | `\\FS01\IT` | ✅ Full Control |
| tom.davis (GRP_Sales) | `\\FS01\Finance` | ❌ Access Denied |
 
> **Why does HR get read-only on Finance?** This mirrors a real-world scenario — HR often needs to view payroll data for compliance purposes but should never be able to modify it. That nuance is intentional.
 
---
 
## Test Users
 
| User | Password | Group | Finance | HR | Sales | IT |
|------|----------|-------|---------|----|-------|----|
| john.smith | P@ssw0rd123! | GRP_IT | Full Control | Full | Full | Full Control |
| sarah.jones | P@ssw0rd123! | GRP_Finance | Modify | Denied | Denied | Denied |
| mike.brown | P@ssw0rd123! | GRP_Finance | Modify | Denied | Denied | Denied |
| lisa.white | P@ssw0rd123! | GRP_HR | Read | Modify | Denied | Denied |
| tom.davis | P@ssw0rd123! | GRP_Sales | Denied | Denied | Modify | Denied |
 
---
 
## NTFS Permission Reference
 
| Flag | Meaning |
|------|---------|
| F | Full Control — can read, write, delete, and change permissions on the folder itself |
| M | Modify — can read, write, and delete files, but cannot change permissions |
| R | Read only — can open and copy files, nothing else |
| (OI) | Object Inherit — files inside the folder also get this rule |
| (CI) | Container Inherit — subfolders inside also get this rule |
 
> **In plain English:** `(OI)(CI)(M)` means "this group can modify files, and that rule trickles down automatically to every file and subfolder created inside."
 
---
 
## Troubleshooting
 
| Issue | Solution |
|-------|----------|
| `configure-lab.ps1` fails on Key Vault step | Run `az login` and confirm your account has the Key Vault Secrets User role |
| Domain join fails | DC01 may still be initialising — wait 2 minutes and re-run from Step 3 |
| Cannot RDP to VMs | Check NSG rules allow port 3389 from your IP (`rdp_source` in tfvars) |
| "Access Denied" on shares | Confirm the user is in the correct group — run `whoami /groups` inside the VM to verify |
| GPO not applying | Run `gpupdate /force` on CLIENT01, wait 2–5 minutes, then check `gpresult /r` |
| Scripts fail to run manually | Set execution policy: `Set-ExecutionPolicy -ExecutionPolicy Bypass -Scope Process` |
| `az vm run-command` times out | VM may have restarted — `configure-lab.ps1` handles this automatically |
| `terraform init` fails | Ensure `backend.tf` has a valid storage account name and the container exists |
 
---
 
## Step 5 — Stop the VMs (Save Money Between Sessions)
 
When you are done for the day but want to keep everything intact, **stop the VMs instead of destroying them.** Stopped (deallocated) VMs do not incur compute charges — think of it like turning off a rented computer without giving it back yet.
 
```bash
az vm stop --ids $(az vm list -g RG-FileServerLab --query "[].id" -o tsv) --no-wait
```
 
When you are ready to pick back up, restart them and give them 3–5 minutes to fully boot:
 
```bash
az vm start --ids $(az vm list -g RG-FileServerLab --query "[].id" -o tsv) --no-wait
```
 
> **Note:** Managed disks and the Key Vault continue to accrue minimal storage costs even when VMs are stopped — but these are a few cents per day, not dollars.
 
---
 
## Teardown
 
When you are completely finished with the lab:
 
```bash
az group delete -n RG-FileServerLab --yes --no-wait
```
 
Only delete the state storage account when you are done with **all** labs that use it:
 
```bash
az group delete -n RG-TerraformState --yes --no-wait
```
 
---
 
## Project Structure
 
```
ntfs-lab-terraform/
├── main.tf                                      # VMs, networking, NSG
├── variables.tf                                 # Input variable definitions
├── outputs.tf                                   # Output values (IPs, Key Vault name)
├── versions.tf                                  # Provider version constraints
├── keyvault.tf                                  # Azure Key Vault + RBAC access
├── backend.tf                                   # Remote state backend (Azure Blob Storage)
├── terraform.tfvars.example                     # Safe template — commit this
├── terraform.tfvars                             # Your real values — DO NOT commit
├── .gitignore                                   # Excludes tfvars, state, .terraform/
├── configure-lab.ps1                            # One-shot automation (run after terraform apply)
└── scripts/
    ├── 00-promote-dc.ps1                        # Promotes DC01 to Domain Controller
    ├── 01-create-ad-users-groups.ps1            # Creates OUs, groups, and test users
    ├── 02-configure-shares-and-permissions.ps1  # SMB shares + NTFS ACLs
    ├── 03-configure-rdp-gpo.ps1                 # GPO for RDP access
    ├── 04-domain-join.ps1                       # Joins FS01 and CLIENT01 to domain
    ├── 05-verify-ad.ps1                         # Automated AD verification
    ├── 05-verify-shares.ps1                     # Automated share + permission verification
    └── 06-add-rdp-users.ps1                     # Grants domain users RDP on CLIENT01
```
 
---
 
## License
 
MIT
