## 1. Lab Title  
Deploying and Managing a Windows Domain Lab in VMware

A hands-on lab to build a Windows Server 2019 domain controller and Windows 10 client in VMware, with snapshotting and OVF/OVA export for easy sharing.

---

## 2. Overview & Objectives  

- **User Story 1:**  
  As a Virtualization Admin, I want to deploy a Windows Server 2019 VM so that I can act as a domain controller.  
  **Objective:** Deploy and configure a Windows Server 2019 VM as an Active Directory Domain Controller (AD DC).

- **User Story 2:**  
  As a Virtualization Admin, I want to deploy a Windows 10 VM and join it to the domain so that I can test Group Policy and AD authentication.  
  **Objective:** Deploy a Windows 10 VM, configure networking, and join it to the lab.local domain.

- **User Story 3:**  
  As a Virtualization Admin, I want to take snapshots of both VMs so that I can revert to a clean state after testing.  
  **Objective:** Create and verify VMware snapshots of both the DC and client VMs.

- **User Story 4:**  
  As a Virtualization Admin, I want to clone the Server VM into an OVF/OVA export so that I can share my lab configuration.  
  **Objective:** Export the Server VM as OVF/OVA and validate the package.

---

## 3. Prerequisites  

- **VMware Workstation Pro** (≥16) *or* **ESXi** host with nested virtualization enabled  
- **Windows Server 2019 ISO** and **Windows 10/11 ISO** (evaluation or trial)  
- **Evaluation license keys** or trial media downloaded  
- **Host machine** with ≥16 GB RAM, ≥4 CPU cores, and ≥100 GB free disk  
- **Tips to validate setup:**  
  - Confirm VMware can create a new VM (File → New Virtual Machine).  
  - Verify nested virtualization: in VM settings under Processor, “Virtualize Intel VT-x/EPT…” is checked.  
  - Ensure ISO images are attached and bootable by mounting them in any test VM.

---

## 4. Architecture Diagram  


---

## 5. Step-by-Step Instructions  

### Step 1: Create the Windows Server 2019 VM

**Action:**  
- In VMware Workstation, go to **File → New Virtual Machine**.  
- Select **Typical**, browse to the Windows Server 2019 ISO.  
- Name it “Server-DC”, store in your lab folder.  
- Set **vCPU: 4**, **RAM: 8 GB**, **Disk: 60 GB (pre-allocate disk)**.  
- Choose **Host-only** network.

**Explanation:**  
- Typical wizard streamlines VM creation.  
- Pre-allocating disk improves performance.  
- Host-only isolates domain traffic from your corporate LAN.

**Validation:**  
- Power on the VM and confirm it boots into the Windows installer.  
- Screenshot: Windows Setup welcome screen.

---

### Step 2: Install Windows Server 2019

**Action:**  
- Run the installer: choose **Standard Installation**, accept EULA, set **Administrator** password.

**Explanation:**  
- Standard Server Core not selected ensures full GUI for DC configuration.

**Validation:**  
- Log in with Administrator and see Server Manager automatically launch.

---

### Step 3: Configure Static Network Settings

**Action:**  
```powershell
New-NetIPAddress -InterfaceAlias "Ethernet0" -IPAddress 192.168.56.10 `
  -PrefixLength 24
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses 192.168.56.10
```

**Explanation:**  
A DC requires a fixed IP so clients and DNS entries remain consistent.

**Validation:**  
Run `ipconfig /all` and verify IPv4 address, DHCP disabled, DNS pointing to self.

---

### Step 4: Promote to Domain Controller

**Action:**  
- In Server Manager, Manage → Add Roles and Features.  
- Select Active Directory Domain Services, complete wizard.  
- After installation, click Promote this server to a domain controller.  
- Create a new forest lab.local, set DSRM password, leave defaults, and reboot.

**Explanation:**  
AD DS role installs the binaries; promotion configures DNS, Global Catalog, and NTDS.

**Validation:**  
After reboot, open Active Directory Users and Computers; verify the lab.local domain exists.

---

### Step 5: Create the Windows 10 VM

**Action:**  
- File → New Virtual Machine, select Windows 10 ISO.  
- Name it “Client-10”, vCPU 2, RAM 4 GB, Disk 40 GB.  
- Use Host-only network.

**Explanation:**  
A smaller footprint VM to simulate a client workstation.

**Validation:**  
VM boots into Windows 10 installer successfully.

---

### Step 6: Install Windows 10

**Action:**  
- Follow installer: choose edition, accept EULA, create local Admin account.

**Explanation:**  
Base OS install to prepare for domain join.

**Validation:**  
Log in and reach the desktop.

---

### Step 7: Configure Client Networking

**Action:**  
```powershell
New-NetIPAddress -InterfaceAlias "Ethernet0" -IPAddress 192.168.56.11 `
  -PrefixLength 24
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses 192.168.56.10
```

**Explanation:**  
Client must use DC for DNS to locate domain services.

**Validation:**  
`nslookup lab.local` returns 192.168.56.10; `ping lab.local` succeeds.

---

### Step 8: Join the Domain

**Action:**  
- Settings → System → About → Join a domain, enter lab.local, provide Administrator credentials.  
- Reboot when prompted.

**Explanation:**  
Domain join creates a computer account in AD and installs the Kerberos client.

**Validation:**  
On login screen, choose Other user, enter `lab\\Administrator` and password, and successfully log in.

---

### Step 9: Take Snapshots of Both VMs

**Action:**  
- In VMware Workstation, select Server-DC, VM → Snapshot → Take Snapshot, name “DC-Clean”.  
- Repeat for Client-10, name “Client-Clean”.

**Explanation:**  
Snapshots capture disk and memory state for quick rollback.

**Validation:**  
Snapshot Manager shows both snapshots; try reverting to confirm functionality.

---

### Step 10: Export the Server VM as OVF/OVA

**Action:**  
- File → Export to OVF, select Server-DC, choose directory, name “Server-DC-Lab”.  
- VMware will generate .ovf (descriptor), .vmdk (disk), and .mf (manifest).  
- Optionally, bundle as .ova via ovftool:
```bash
ovftool Server-DC-Lab.ovf Server-DC-Lab.ova
```

**Explanation:**  
OVF is an open packaging format; OVA is its single-file archive.

**Validation:**  
Confirm .ovf/.vmdk or .ova exists in target folder, size matches VM disk.

---

## 6. Troubleshooting & FAQs

### **Error 1:** VM won’t boot from ISO  
- **Cause:** Boot order wrong or ISO not connected  
- **Fix:** In VM settings, ensure CD/DVD mounted & “Force BIOS setup on next boot” is checked.

### **Error 2:** DCPromo fails with DNS lookup errors  
- **Cause:** Server’s DNS pointing elsewhere  
- **Fix:** Set DNS to 127.0.0.1 or its own static IP before AD DS role installation.

### **Error 3:** Domain join hangs or “domain not found”  
- **Cause:** Misconfigured DNS or network isolation  
- **Fix:** Verify client DNS points to DC (`ipconfig /all`), ping DC IP, adjust host-only settings.

### **Error 4:** Snapshot creation fails (“insufficient space”)  
- **Cause:** Host datastore nearly full  
- **Fix:** Free up space or increase virtual disk datastore capacity; consider thin provisioning.

### **Error 5:** OVF export incomplete (missing files)  
- **Cause:** Files in use by running VM  
- **Fix:** Power off the VM completely before export; ensure no open handles to VMDKs.


7. Expert Insights
**Security Best Practices:**
- After promotion, run Windows Update and patch OS immediately.
- Restrict RDP to specific networks and use strong complex passwords.
- Enable BitLocker on data drives to protect VM disks at rest.

**Performance Tuning:**
- Use Paravirtual SCSI adapters for production workloads to reduce CPU overhead.
- Pre-allocate and align disks to OS block size (1 MB) to avoid fragmentation.
- Install VMware Tools for optimized drivers and heartbeat integration.

**Pro Tips:**
- Automate with PowerCLI: Script snapshotting, exports, and domain promotion for repeatable labs.
- Template Golden Image: Once a VM is clean and patched, convert to a template for rapid deployments.
- Network Isolation: Leverage multiple host-only networks to segment DC, clients, and test services.
- Use Linked Clones: Save storage and time by creating linked clones of your DC for multi-site scenarios.

## 8. Knowledge-Retention Activities
### Reflection Questions
1. Why is it critical to assign a static IP address to a domain controller rather than relying on DHCP?

2. How do VMware snapshots work under the hood, and what are the performance trade-offs of keeping snapshots long term?

3. What are the differences between exporting as OVF versus OVA, and in which scenarios would you choose one over the other?

### Quiz

1. **Which VMware network type isolates VMs from the physical LAN but allows inter-VM communication?**

   - A. Bridged  
   - B. NAT  
   - C. Host-only  
   - D. Custom  

   **Answer:** C. Host-only

2. **What PowerShell cmdlet promotes a server to a domain controller?**

   - A. Install-ADDSForest  
   - B. Install-WindowsFeature AD-Domain-Services  
   - C. Add-DnsServerPrimaryZone  
   - D. Install-ADDSDomain  

   **Answer:** A. Install-ADDSForest

3. **Which TCP port must be open for initial LDAP communication with a domain controller?**

   - A. 88  
   - B. 135  
   - C. 389  
   - D. 636  

   **Answer:** C. 389

4. **What file extension is used for a single-file archive containing an OVF package?**

   - A. .ovf  
   - B. .ova  
   - C. .vmdk  
   - D. .mf  

   **Answer:** B. .ova

5. **When taking a VMware snapshot, what two states are captured?**

   - A. Disk state only  
   - B. Memory and CPU state only  
   - C. Disk and memory state  
   - D. Network traffic  

   **Answer:** C. Disk and memory state

### Spaced-Repetition Plan

**Day 1 Review:**  
- Task: Review static IP configuration for domain controllers in VMware.  
- Prompt: “Explain why DCs need static IPs and how to configure them in VMware.”

**Week 1 Review:**  
- Task: Review domain controller promotion steps and PowerShell commands.  
- Prompt: “Walk through the steps and PowerShell commands to promote a DC.”

**Month 1 Review:**  
- Task: Review VM export and import process.  
- Prompt: “Demonstrate exporting a VM as OVA and re-importing it in VMware.”