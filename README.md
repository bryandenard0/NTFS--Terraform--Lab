🗂️ NTFS & Active Directory Lab — Azure Terraform
A fully automated home lab that spins up a real Windows domain environment in Azure — the same kind of setup you'd find in a corporate office. Built to practice NTFS permissions, Active Directory administration, and Group Policy in a safe, disposable cloud environment.

No experience with enterprise IT? No problem. This README explains everything in plain English alongside the technical terms so you know what is happening and why.


🧠 What This Lab Does (Plain English)
Imagine a small company with four departments: Finance, HR, Sales, and IT. Each department has its own shared folder on a file server, and employees should only be able to access the folders that belong to their department.
This lab builds that entire setup automatically:

A Domain Controller (the "boss" server that manages who can log in and what they can access)
A File Server with department folders and locked-down permissions
A Client PC where you log in as a test employee and experience the permissions firsthand
User accounts and groups for each department
Group Policy to enforce Remote Desktop access rules

All of it runs in Microsoft Azure and is built with Terraform — meaning you can spin it up, practice, and tear it all down with a few commands.

🏗️ Architecture
Azure Resource Group
│
├── DC01  (Windows Server 2022 — Domain Controller)
│     └── Active Directory domain: lab.local
│     └── DNS server
│     └── Group Policy management
│
├── FS01  (Windows Server 2022 — File Server)
│     └── C:\Shares\Finance   → GRP_Finance (Modify), GRP_IT (Full Control)
│     └── C:\Shares\HR        → GRP_HR (Modify), GRP_IT (Full Control)
│     └── C:\Shares\Sales     → GRP_Sales (Modify), GRP_IT (Full Control)
│     └── C:\Shares\IT        → GRP_IT (Full Control)
│
├── CLIENT01  (Windows — workstation for testing)
│     └── Joined to lab.local domain
│     └── Domain Users granted RDP access
│
└── Azure Key Vault
      └── Stores admin credentials securely

👥 Users & Groups
UserDepartment GroupShare Accessjohn.smithGRP_ITAll shares (Full Control)sarah.jonesGRP_FinanceFinance (Modify)mike.brownGRP_FinanceFinance (Modify)lisa.whiteGRP_HRHR (Modify)tom.davisGRP_SalesSales (Modify)

NTFS Permissions explained: "Modify" means you can read, write, and delete files in that folder. "Full Control" means you can also change permissions on the folder itself — that's the IT admin level.


🔐 NTFS Permission Matrix
GroupFinanceHRSalesITGRP_Finance✅ Modify❌❌❌GRP_HR👁️ Read✅ Modify❌❌GRP_Sales❌❌✅ Modify❌GRP_IT✅ Full Control✅ Full Control✅ Full Control✅ Full Control

Why does HR have Read on Finance? This is intentional — in many companies HR needs to view payroll/finance data but not change it. This mirrors that real-world scenario.


🛠️ Tech Stack
ToolWhat It DoesTerraformProvisions all Azure infrastructure as code — VMs, networking, Key VaultAzureCloud provider hosting all VMsWindows Server 2022Operating system for DC01 and FS01Active Directory Domain Services (AD DS)Microsoft's directory service — manages users, groups, computersNTFSThe Windows file system that enforces folder-level permissionsSMB (Server Message Block)The network protocol that lets CLIENT01 access \FS01\ShareNameGroup Policy (GPO)Enforces settings across the domain (e.g., who can RDP into machines)Azure Key VaultStores credentials securely — no passwords hardcoded in scriptsPowerShellAutomates all post-deployment configuration

📋 Prerequisites

Terraform installed
Azure CLI installed and logged in (az login)
An active Azure subscription
Sufficient quota for 3x Standard_B2ms VMs in your chosen region


🚀 Deployment
1. Clone the repo
bashgit clone https://github.com/YOUR-USERNAME/ntfs-lab-terraform.git
cd ntfs-lab-terraform
2. Initialize and deploy infrastructure
bashterraform init
terraform plan
terraform apply
This creates all Azure resources — VMs, VNet, Key Vault, NSGs. Takes ~5 minutes.
3. Configure the lab
powershell.\scripts\configure-lab.ps1 -KeyVaultName "your-keyvault-name"
This runs all configuration steps automatically (~15 minutes):

Promotes DC01 to Domain Controller
Creates OUs, groups, and users in Active Directory
Joins FS01 and CLIENT01 to the domain
Creates SMB shares and applies NTFS ACLs
Configures Group Policy
Runs automated verification

4. Connect and test
1. Get CLIENT01's public IP:
   az vm list-ip-addresses --name CLIENT01 --output table

2. RDP into CLIENT01:
   mstsc /v:<CLIENT01-public-IP>

3. Log in as a test user:
   Username: LAB\sarah.jones
   Password: P@ssw0rd123!

4. Open File Explorer and try accessing:
   \\FS01\Finance   ← sarah.jones can access (GRP_Finance)
   \\FS01\HR        ← Access denied (not in GRP_HR)

🧪 What to Practice
Permissions Testing

Log in as different users and observe what they can and can't access
Try to access a share you don't belong to — confirm you get "Access Denied"
Log in as john.smith (IT) and confirm Full Control on all shares

Active Directory Administration

Open Active Directory Users and Computers on DC01
Add a new user to a group and verify their access changes
Move a computer object between OUs

Group Policy

Open Group Policy Management on DC01
View the RDP GPO linked to the Lab Computers OU
Create a new GPO and link it to an OU

NTFS Deep Dive

Right-click a share folder on FS01 → Properties → Security tab
View the Access Control List (ACL) — these are the actual permission entries
Use icacls C:\Shares\Finance in PowerShell to view permissions in the CLI


🧹 Teardown
When you're done, destroy everything to avoid Azure charges:
bashterraform destroy

📁 Project Structure
ntfs-lab-terraform/
├── main.tf                  # Core infrastructure
├── variables.tf             # Input variables
├── outputs.tf               # Output values (IPs, resource names)
├── configure-lab.ps1        # Wrapper script — orchestrates all steps
└── scripts/
    ├── 00-promote-dc.ps1            # Installs AD DS, promotes DC01
    ├── 01-create-ad-users-groups.ps1 # Creates OUs, groups, users
    ├── 02-configure-shares-and-permissions.ps1  # SMB shares + NTFS ACLs
    ├── 03-configure-rdp-gpo.ps1     # Creates and links RDP Group Policy
    ├── 04-domain-join.ps1           # Joins FS01 and CLIENT01 to domain
    ├── 05-verify-ad.ps1             # Automated AD verification
    ├── 05-verify-shares.ps1         # Automated share/permission verification
    └── 06-add-rdp-users.ps1         # Grants Domain Users RDP access

🔍 Key Concepts Explained
What is a Domain Controller?
Think of it like the front desk of a large office building. Everyone who wants to get in has to check in here first. The DC verifies your identity and tells the rest of the network what you're allowed to do.
What is NTFS?
NTFS (New Technology File System) is how Windows keeps track of files AND who is allowed to access them. Every file and folder has an invisible "guest list" (called an ACL — Access Control List) that Windows checks before letting you in.
What is Active Directory?
Active Directory is Microsoft's system for managing users, computers, and permissions across an entire organization. Instead of setting permissions on every individual machine, you manage everything from one place — the Domain Controller.
What is a GPO?
A Group Policy Object is like a company rulebook that gets automatically pushed to every computer in the domain. You write the rule once on the DC, and it applies everywhere — no need to configure each machine individually.

📌 Notes

All passwords in this lab are intentionally simple (P@ssw0rd123!) — do not use in production
The lab domain lab.local is isolated and not routable to the public internet
NSG rules restrict RDP access — review main.tf to lock down to your IP if desired
The GPO script (03-configure-rdp-gpo.ps1) may require re-running if encoding issues occur during deploymen
