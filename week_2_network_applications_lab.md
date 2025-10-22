# Week 2 — Network Applications (QHO443): Workgroup File Sharing — Slide-by-Slide Lab Pack

> **House style:** mirrors your *DiscoverHealthPartB.markdown* — Introduction → What you’ll learn/do → numbered Exercises with CLI, Troubleshooting, and a Summary.

## Introduction
In this lab you will configure a peer-to-peer **Windows Workgroup** (or macOS host) and connect from a **Linux VM** using SMB/CIFS. You will create local users and groups, set **Share** and **NTFS** (or filesystem) permissions, map a network drive, verify effective access, and capture evidence for your **PLR**.

### What you’ll learn
- Workgroup vs Domain basics; SMB/CIFS sharing across Windows/macOS/Linux.
- How **Share** permissions and **NTFS** permissions combine (**most restrictive wins**).
- Creating local **users** and **groups** and applying **least privilege**.
- Mapping shares from Windows and mounting from Linux (`cifs`), plus `smbclient`.
- Troubleshooting discovery, firewall profile, services, name vs IP, SMB version.
- Writing PLR: **Intro/Method/Summary** with annotated screenshots (Assessment Clinic).

---

## Read this first — simple definitions (beginner)

- **Host** = your real laptop that will **share the folder**. Pick **one**: your **Windows laptop** *or* your **Mac**.
- **Guest / VM** = the **Linux virtual machine** running inside VirtualBox/Parallels/UTM. This acts as the **second computer**.
- **Which computer do I click on?**  
  • If your **host is Windows**, do the **Workgroup**, **Discovery/Sharing**, **User/Group**, and **Share/NTFS** steps **on the Windows host only**.  
  • If your **host is Mac**, **skip** the Windows Workgroup bits and do the **macOS File Sharing** steps **on the Mac host**.  
  • All **Linux commands** are done **inside the Linux VM**.
- **Network mode for the VM**: set to **Bridged** if possible (so the VM gets a normal IP like `192.168.x.x`). NAT can work but IP testing is easier with Bridged.

## Slide Map & Exercises (answered)

### Activity 1 — Configure a Windows Workgroup & Verify Connectivity
**Goal (why):** Make two Windows systems see each other in a **Workgroup**, then prove it with **ping**, **net view**, and service checks. This confirms the network is ready before we add permissions. 
**Do this on:** **Windows HOST** (your real Windows laptop). **Not** in the Linux VM.  
**If your host is a Mac, skip this and use *Activity M — macOS host* below.**
**From slides:** Rename Workgroup → enable Network Discovery/File Sharing → `ping`/`net view` → check services → **Q:** *If access fails, what could be the reason?*

**Steps (Windows host + a second Windows/VM):**
1) **Workgroup**
   - Settings → System → About → **Rename this PC (Advanced)** → **Change…** → Workgroup: `NetAppsLab` → OK → restart.
2) **Discovery & Sharing**
   - Control Panel → Network and Sharing Centre → **Change advanced sharing settings** → turn **ON**: *Network Discovery* and *File and printer sharing*.
3) **Connectivity tests (on each PC)**
   ```bat
   ping <OtherComputerName>
   net view \\<OtherComputerName>
   ```
4) **SMB services (PowerShell)**
   ```powershell
   Get-Service LanmanServer
   Get-Service LanmanWorkstation
   ```

**Answer to the slide question — “If access fails, what could be the reason?”**
- Network profile is **Public** (sharing blocked); set to **Private**.
- **Network Discovery/File sharing** not enabled.
- **LanmanServer/LanmanWorkstation** stopped/disabled.
- Name resolution failing; try **IP address** instead of hostname.
- Machines on **different subnets/VLANs** (or VM not in **Bridged** mode).
- Local firewall/AV blocking SMB; allow File/Printer sharing or try Private network.
- Wrong credentials or account disabled/locked.

**Expected outcome:** both systems can ping and list each other’s shares.

---

### Activity 2 — Create Local Users/Groups and Apply Permissions
**Goal (why):** Build **controlled access** to a shared folder using a **least‑privilege** user and group. Apply **Share + NTFS** permissions and verify with **Effective Access** so we know exactly what the user can do. 
**Do this on:** **Windows HOST** (your real Windows laptop). **Not** in the Linux VM.  
**Mac hosts:** you’ll set access in macOS File Sharing instead; see *Activity M — macOS host* below.
**From slides:** Gap-fill warm-up → create `User1` and group → share **C:\\SharedData** → set Share + NTFS → verify Effective Access.

**Gap-Fill Answers (from the slide prompts):**
- **W______** = **Workgroup** — Local Users/Groups are ideal for **Workgroup** environments.
- **S______** = **Share** permissions — control **over the network**.
- **N______** = **NTFS** permissions — control **locally and remotely**.
- Most **r________** = **restrictive** permission wins when combining Share + NTFS.
- Principle of **L______** = **Least** Privilege — give only the minimum access.

**Do the configuration (Windows):**
1) **Create a user (GUI):** `Win + R` → `compmgmt.msc` → Local Users and Groups → Users → **New User…**
   - Username: `User1` (password of your choice).
2) **Create a group + add user (Admin CMD):**
   ```bat
   net localgroup ProjectTeam /add
   net localgroup ProjectTeam User1 /add
   ```
3) **Create + share the folder:**
   - Make folder **C:\\SharedData**.
   - Properties → **Sharing → Advanced Sharing…** → tick **Share this folder** → **Permissions…**
     - **Remove** *Everyone* → **Add** `ProjectTeam` → **Allow**: **Change**, **Read**.
4) **NTFS permissions (Security tab):**
   - **Add** `ProjectTeam` → **Allow**: **Modify**, **Read & execute**, **List**, **Read**.
5) **Verify Effective Access:**
   - Properties → **Security → Advanced → Effective Access** → **Select a user…** → `User1` → *View* → should show **Modify**.

**Expected outcome:** `User1` (via `ProjectTeam`) has **Modify** on **C:\\SharedData**; Share+NTFS obey **most restrictive wins**.

---

### Activity 3 — Map a Network Drive, Test Access, Troubleshoot
**Goal (why):** Show that authorised users can **create/edit/delete** files, and that removing access leads to **Access Denied**. This proves your permissions really work. 
**Do this on:** the **second computer**. In our setup that’s the **Linux VM**. (If you have a real second Windows PC, you can do the same steps there.)
**From slides:** Login as `User1` → map drive → CRUD test → remove from group → Access Denied → troubleshooting commands.

**Steps (second Windows PC/VM):**
1) **Sign in** as `User1` (or supply those creds when mapping).
2) **Map the drive** (replace host name/IP as needed):
   ```bat
   net use Z: \\PC01\SharedData /user:User1 YourPassword
   ```
3) **Test access:** create a text file, edit it, delete it.
4) **Prove denial:** on the host run:
   ```bat
   net localgroup ProjectTeam User1 /delete
   ```
   Then try to create/delete again from Z: — expect **Access Denied**.
5) **Troubleshooting (examples):**
   ```powershell
   Get-Service LanmanServer
   ```
   ```bat
   net use * /delete
   icacls "C:\\SharedData"
   ```

**Expected outcome:** successful mapping for `User1` while in the group; **denied** after removal.

---

### Extension / Stretch Task — Use Linux as the Client
**Goal (why):** Access the Windows share from **Linux** and spot any cross‑platform issues (SMB version, credentials). 
**From slides:** `smbclient -L //<windows-ip>/ -U <username>` and note issues.

**Linux VM (Ubuntu/Debian):**
```bash
sudo apt update && sudo apt install -y cifs-utils smbclient
smbclient -L //192.168.1.23/ -U User1
sudo mkdir -p /mnt/shareddata
sudo mount -t cifs //192.168.1.23/SharedData /mnt/shareddata -o username=User1,vers=3.0
cd /mnt/shareddata && echo "hello" > test.txt && cat test.txt && rm test.txt
```

---

## Assessment Clinic (slides) — Questions Answered

### Clinic Activity 1 — PLR Structure (True/False)
> Decide T/F based on the guidance.

1) “Your PLR should **only include screenshots**; detailed explanations are **not required**.” → **False**  
2) “The **Introduction** should briefly explain the **purpose** of the lab and its **relevance**.” → **True**  
3) “The **Summary** should reflect on what worked, challenges faced, and possible improvements.” → **True**

### Clinic Activity 2 — Write a Strong Technical Introduction (example to copy/adapt)
> **Example (3–4 sentences):**  
“This lab configured a peer-to-peer Windows Workgroup to enable local resource sharing without a centralised domain controller. We created a least-privilege local user/group and applied both Share and NTFS permissions, then validated effective access from a second client. We demonstrated that the most restrictive permission wins by removing the user’s group membership to produce Access Denied. This approach reflects common small-network scenarios where basic access control is required without deploying Active Directory.”

---

## Evidence to capture for your PLR
- Workgroup name set to **NetAppsLab** (both PCs) and **Discovery/Sharing ON**.
- `ping` + `net view` + `Get-Service` results.
- User/Group creation and **Share + NTFS** screenshots.
- **Effective Access** for `User1` = **Modify**.
- Successful `net use Z:` mapping and file CRUD.
- **Access Denied** after removing `User1` from `ProjectTeam`.
- Linux `smbclient -L` + mounted share and CRUD.

---

## Troubleshooting quick table
- **Use IP** instead of name if browsing fails.  
- Ensure Windows network **Private**; Discovery/Sharing ON.  
- `LanmanServer` running; firewall allows File/Printer sharing.  
- VM on **Bridged** network; same subnet.  
- Clear cached mappings (`net use * /delete`).  
- Check NTFS via `icacls "C:\\SharedData"`; SMB version with `vers=3.0` when mounting.

---

## Summary
You configured cross-platform SMB file sharing, applied least privilege with Share + NTFS, validated effective access, and documented evidence for your PLR. You also answered all slide questions, including the gap-fill and T/F quiz, and provided a model introduction for your write-up.

---

### Activity M — macOS host (use these steps if your host is a Mac)
**Goal (why):** Provide the **same controlled sharing** when your **Mac** is the host, then confirm Linux can read/write and can be denied when permissions are lowered. 
**Do this on:** **macOS HOST** (your real Mac laptop). **Not** in the Linux VM.

1) **Turn on File Sharing (SMB)**  
   System Settings → **General** → **Sharing** → turn **File Sharing** **ON**.  
   Click the **ⓘ / Options…** button and make sure **SMB** is enabled.

2) **Add the folder to share**  
   In **Shared Folders**, click **+** and add a folder (example: `~/SharedData`).  
   In the **Users** list on the right, give your login user **Read & Write**.

   > Optional: create a separate standard user for the lab: System Settings → **Users & Groups** → **Add User…** → *Standard*. You can use that account for SMB instead of your main user.

3) **Find your Mac’s IP address**  
   System Settings → **Wi‑Fi** → your network → **Details** → **IP Address** (e.g., `192.168.1.50`).  
   Or Terminal: `ipconfig getifaddr en0` (Wi‑Fi is usually `en0`).

4) **(Optional) Set SMB Workgroup name**  
   This is usually **not required**. If your teacher wants a specific workgroup name: open **Directory Utility** → **SMB** → set **Workgroup** to `NetAppsLab`.

5) **Connect from the Linux VM**  
   In the Linux VM, install tools if needed:  
   ```bash
   sudo apt update && sudo apt install -y cifs-utils smbclient
   ```
   List shares on the Mac:  
   ```bash
   smbclient -L //192.168.1.50/ -U <your-mac-username>
   ```
   Mount the share (change the IP and folder name if different):  
   ```bash
   sudo mkdir -p /mnt/macshare
   sudo mount -t cifs //192.168.1.50/SharedData /mnt/macshare -o username=<your-mac-username>,vers=3.0
   ```
   Test create/read/delete:  
   ```bash
   cd /mnt/macshare
   echo "hello from linux" > test.txt
   cat test.txt
   rm test.txt
   ```

6) **Prove Access Denied (Mac)**  
   Back on the Mac: in **File Sharing**, change that user’s permission to **Read only** (or remove the user).  
   In the Linux VM, try to create/delete again — it should **fail**.

> ✅ If your **host is a Mac**, you can **skip Activities 1 & 2** (those are Windows‑only) and do **Activity 3** + **Activity M** instead.

