# 1. Lab Title
Deploying and Managing a Windows Domain Lab in Hyper-V

A hands-on lab to build a Windows Server 2019 domain controller and Windows 10 client in Hyper-V, with checkpoints and VM export for easy sharing.

# 2. Overview & Objectives

**User Story 1:**
As a Virtualization Admin, I want to deploy a Windows Server 2019 VM so that I can act as a domain controller.
- **Objective:** Deploy and configure a Windows Server 2019 VM as an Active Directory Domain Controller (AD DC).

**User Story 2:**
As a Virtualization Admin, I want to deploy a Windows 10 VM and join it to the domain so that I can test Group Policy and AD authentication.
- **Objective:** Deploy a Windows 10 VM, configure networking, and join it to the lab.local domain.

**User Story 3:**
As a Virtualization Admin, I want to take checkpoints of both VMs so that I can revert to a clean state after testing.
- **Objective:** Create and verify Hyper-V checkpoints of both the DC and client VMs.

**User Story 4:**
As a Virtualization Admin, I want to export the Server VM so that I can share my lab configuration.
- **Objective:** Export the Server VM via Export-VM and validate the package.

# 3. Prerequisites

- **Hyper-V host:** Windows 10/11 Pro or Windows Server with Hyper-V role enabled (nested virtualization enabled if running in cloud)
- **Virtual Switch:** Create an Internal (or Private) switch named LabSwitch via Hyper-V Manager → Virtual Switch Manager
- **Windows Server 2019 ISO** and **Windows 10/11 ISO** (evaluation or trial)
- **Evaluation license keys** or trial media downloaded
- **Host machine** with ≥16 GB RAM, ≥4 CPU cores, and ≥100 GB free disk

**Tips to validate setup:**
- Confirm Hyper-V is installed: run `Get-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V` returns Enabled.
- Verify LabSwitch exists: `Get-VMSwitch –Name LabSwitch`.
- Ensure ISOs are accessible at known paths (e.g. `C:\ISOs\WindowsServer2019.iso`).

---

# 4. Architecture Diagram

```
+-----------------------------------------+
|             Hyper-V Host                |
|                                         |
|  +---------------+   +---------------+  |
|  | Server-DC VM  |   | Client-10 VM  |  |
|  | (Win Srv 2019)|   | (Win 10/11)   |  |
|  |  vCPU: 4      |   |  vCPU: 2      |  |
|  |  RAM: 8 GB    |   |  RAM: 4 GB    |  |
|  |  Disk: 60 GB  |   |  Disk: 40 GB  |  |
|  |  IP: 192.168.56.10/24          |  |
|  |  DNS: self (192.168.56.10)    |  |
|  +---------------+   +---------------+  |
|           \           /                 |
|            \         /                  |
|             \       /                   |
|         LabSwitch (Internal)            |
+-----------------------------------------+
```

# 5. Step-by-Step Instructions

## Step 1: Create the Windows Server 2019 VM
**Action:**
```powershell
New-VM -Name Server-DC `
  -MemoryStartupBytes 8GB `
  -Generation 2 `
  -NewVHDPath C:\VMs\Server-DC.vhdx `
  -NewVHDSizeBytes 60GB `
  -SwitchName LabSwitch
Set-VMProcessor -VMName Server-DC -Count 4
Add-VMDvdDrive -VMName Server-DC -Path C:\ISOs\WindowsServer2019.iso
```
**Explanation:**
- Creates a Gen 2 VM named “Server-DC” with 4 vCPUs, 8 GB RAM, a fixed 60 GB VHDX, and attaches it to LabSwitch.
- Attaches the Windows Server ISO to the virtual DVD drive for installation.

**Validation:**
- Run `Get-VM -Name Server-DC` → Status should be Off and VM settings match.
- Confirm VHDX and DVD drive paths via `Get-VMHardDiskDrive` & `Get-VMDvdDrive`.

## Step 2: Install Windows Server 2019
**Action:**
- In Hyper-V Manager, Start the “Server-DC” VM.
- Connect to the console and proceed through the Windows installer:
  - Choose Standard Installation (Desktop Experience).
  - Accept license, create Administrator password.

**Explanation:**
- Desktop Experience provides the full GUI needed to configure AD DS via Server Manager.

**Validation:**
- After install and reboot, log in as Administrator. Server Manager should auto-launch.

## Step 3: Configure Static Network Settings
**Action (inside guest):**
```powershell
New-NetIPAddress `
  -InterfaceAlias "Ethernet" `
  -IPAddress 192.168.56.10 `
  -PrefixLength 24
Set-DnsClientServerAddress `
  -InterfaceAlias "Ethernet" `
  -ServerAddresses 192.168.56.10
```

**Explanation:**
- A DC requires a fixed IP so DNS records and client lookups remain stable.

**Validation:**
- `ipconfig /all` shows IPv4 = 192.168.56.10/24, DHCP disabled, DNS = 192.168.56.10.

## Step 4: Promote to Domain Controller
**Action (GUI):**
- In Server Manager → Manage → Add Roles and Features → install Active Directory Domain Services.
- Click Promote this server to a domain controller.
- Select Add a new forest, set Root domain name = lab.local, specify DSRM password, keep defaults, then Install.

**Alternatively (PowerShell):**
```powershell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
Import-Module ADDSDeployment
Install-ADDSForest `
  -DomainName "lab.local" `
  -SafeModeAdministratorPassword (ConvertTo-SecureString "P@ssw0rd!" -AsPlainText -Force) `
  -InstallDNS
```

**Explanation:**
- Installs AD DS binaries, creates DNS zone, Global Catalog, and the first domain controller in the new forest.

**Validation:**
- After reboot, open Active Directory Users and Computers → verify lab.local domain exists.

## Step 5: Create the Windows 10 VM
**Action:**
```powershell
New-VM -Name Client-10 `
  -MemoryStartupBytes 4GB `
  -Generation 2 `
  -NewVHDPath C:\VMs\Client-10.vhdx `
  -NewVHDSizeBytes 40GB `
  -SwitchName LabSwitch
Set-VMProcessor -VMName Client-10 -Count 2
Add-VMDvdDrive -VMName Client-10 -Path C:\ISOs\Windows10.iso
```

**Explanation:**
- Builds a smaller Gen 2 VM for the client, attached to the same Internal switch.

**Validation:**
- `Get-VM -Name Client-10` → matches specs, status Off.

## Step 6: Install Windows 10
**Action:**
- Start and connect to “Client-10”.
- Complete the Windows 10 installer: select edition, accept EULA, create a local Admin account.

**Explanation:**
- Prepares the client OS for domain join.

**Validation:**
- Log in to the desktop as the local Admin.

## Step 7: Configure Client Networking
**Action (inside guest):**
```powershell
New-NetIPAddress `
  -InterfaceAlias "Ethernet" `
  -IPAddress 192.168.56.11 `
  -PrefixLength 24
Set-DnsClientServerAddress `
  -InterfaceAlias "Ethernet" `
  -ServerAddresses 192.168.56.10
```

**Explanation:**
- Client must use the DC for DNS to locate domain services.

**Validation:**
- `nslookup lab.local` → returns 192.168.56.10
- `ping lab.local` → successful replies.

## Step 8: Join the Domain
**Action (GUI):**
- Settings → System → About → Join a domain → enter lab.local, provide Administrator credentials, then reboot.
**Or (PowerShell):**
```powershell
Add-Computer `
  -DomainName "lab.local" `
  -Credential (Get-Credential lab\Administrator) `
  -Restart
```

**Explanation:**
- Adds the workstation to AD, installs Kerberos client, and creates a computer account.

**Validation:**
- On reboot, at login screen choose Other user, enter lab\Administrator, and succeed.

## Step 9: Take Checkpoints of Both VMs
**Action:**
```powershell
Checkpoint-VM -Name Server-DC -SnapshotName DC-Clean
Checkpoint-VM -Name Client-10 -SnapshotName Client-Clean
```

**Explanation:**
- Checkpoints capture VM configuration, disk & memory state for rollback.

**Validation:**
- In Hyper-V Manager → right-click VM → Checkpoints shows DC-Clean and Client-Clean.
- Test revert: Apply a checkpoint and confirm VM state resets.

## Step 10: Export the Server VM
**Action:**
```powershell
Stop-VM -Name Server-DC
Export-VM -Name Server-DC -Path C:\LabExports\Server-DC
```

**Explanation:**
- Exports VM files (configuration, VHDX, snapshots) into a folder for sharing or archival.

**Validation:**
- Confirm folder C:\LabExports\Server-DC contains .vmc, .vhdx, and checkpoint files.
- Test import on another Hyper-V host via Import Virtual Machine.

# 6. Troubleshooting & FAQs

### Error 1: VM fails to power on
- **Cause:** Hyper-V role not enabled or BIOS virtualization off
- **Fix:** Enable Hyper-V: `Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V` and reboot; enable VT-x/AMD-V in BIOS.

### Error 2: “LabSwitch” not available
- **Cause:** Virtual Switch missing or mis-named
- **Fix:** In Hyper-V Manager → Virtual Switch Manager, create an Internal switch named LabSwitch.

### Error 3: Installer won’t boot
- **Cause:** DVD drive not first in firmware boot order
- **Fix:** `Set-VMFirmware -VMName Server-DC -FirstBootDevice (Get-VMDvdDrive -VMName Server-DC)`

### Error 4: Domain join “not found”
- **Cause:** DNS misconfigured on client
- **Fix:** Reconfigure client DNS to 192.168.56.10; confirm via `ipconfig /all`.

### Error 5: Checkpoint creation errors
- **Cause:** Insufficient disk space or VM replication enabled
- **Fix:** Free up storage, disable replication, then retry `Checkpoint-VM`.

### Error 6: Export-VM fails
- **Cause:** VM is running or snapshots open
- **Fix:** Stop VM (`Stop-VM`), then rerun `Export-VM`.

# 7. Expert Insights

**Security Best Practices:**
- Use Production Checkpoints (VSS-based) for data consistency on DCs.
- Enable Shielded VMs or BitLocker on VHDX files to encrypt disks at rest.
- Limit Hyper-V host management ports via Windows Firewall or Admin ACLs.

**Performance Tuning:**
- Use Dynamic Memory for client VMs to optimize RAM usage.
- Ensure VMQ (Virtual Machine Queue) and RSS are enabled on virtual NICs for throughput.
- Align VHDX to 1 MB logical sectors; Hyper-V does this by default on Gen 2.

**Pro Tips:**
- Leverage Differencing Disks against a golden “baseline” image for rapid VM clones.
- Automate VM lifecycle (creation, checkpoint, export) with PowerShell Desired State Configuration.
- Document Hyper-V host health via `Measure-VM` and `Get-VMIntegrationService` reports.

# 8. Knowledge-Retention Activities

## Reflection Questions
1. Why must a domain controller use a static IP, and what issues arise if it doesn’t?
2. How do Hyper-V Production vs. Standard checkpoints differ, and when should each be used?
3. Describe the steps Hyper-V takes under the hood when exporting a VM.

## Quiz
1. **Which PowerShell cmdlet creates a new Hyper-V VM?**
   - A. New-VHD
   - B. New-VM
   - C. Add-VMHardDiskDrive
   - D. New-Item
   **Answer:** B. New-VM

2. **What PowerShell command attaches an ISO as a DVD drive?**
   - A. Add-VHD
   - B. Set-VMDvdDrive
   - C. Add-VMDvdDrive
   - D. Mount-DiskImage
   **Answer:** C. Add-VMDvdDrive

3. **Which switch type isolates VMs from the host’s physical LAN but allows inter-VM traffic?**
   - A. External
   - B. Internal
   - C. Private
   - D. NAT
   **Answer:** B. Internal

4. **What port must be open for LDAP communication?**
   - A. 88
   - B. 389
   - C. 445
   - D. 636
   **Answer:** B. 389

5. **Which cmdlet exports a Hyper-V VM for sharing?**
   - A. Save-VM
   - B. Export-VM
   - C. Backup-VM
   - D. Copy-VMFile
   **Answer:** B. Export-VM

## Spaced-Repetition Plan

**Day 1 Review:**
- Task: Review the process and PowerShell commands to create LabSwitch.
- Prompt: “Explain the process and PowerShell commands to create LabSwitch.”

**Week 1 Review:**
- Task: Review installing AD DS via GUI and PowerShell on the DC.
- Prompt: “Walk through installing AD DS via GUI and PowerShell on the DC.”

**Month 1 Review:**
- Task: Review checkpoint creation and VM export/import in Hyper-V.
- Prompt: “Demonstrate checkpoint creation and VM export/import in Hyper-V.”