# Active Directory Lab

A hands-on Active Directory lab built on Windows Server 2025, demonstrating domain configuration, OU structure, Group Policy deployment, and shared folder permission management.

## Environment
- **Hypervisor:** VMware Workstation
- **Domain Controller:** Windows Server 2025
- **Client:** Windows 11
- **Domain:** ducklab.local

---

## 1. Network Setup

Windows 11 was joined to the ducklab.local domain by:
1. Setting the Windows 11 DNS server to the Windows Server IP address
2. Navigating to System Properties → Change → Domain → entering ducklab.local
3. Authenticating with domain administrator credentials

The domain controller acts as the DNS server for the domain — Active Directory requires DNS to locate domain services.

![description](https://github.com/nullmarch/active-directory-lab/blob/main/Screenshots/ducklab.local.domain.png)

---

## 2. Active Directory Structure

Users and computers are organized into a dedicated OU structure:

![description](https://github.com/nullmarch/active-directory-lab/blob/main/Screenshots/ADUC.PNG)

---

## 3. GPO 1 — Screen Lock Policy

A Group Policy Object was created and linked to the Lab Users OU to enforce screen locking after inactivity.

![description](https://github.com/nullmarch/active-directory-lab/blob/main/Screenshots/Screen_Lock_Policy_GPO.PNG)


### Computer Configuration
**Path:** Computer Configuration → Policies → Windows Settings → Security Settings → Local Policies → Security Options

- **Interactive logon: Machine inactivity limit** — set to 300 seconds

This setting locks the workstation at the OS level and cannot be bypassed by the user.

![description](https://github.com/nullmarch/active-directory-lab/blob/main/Screenshots/GPO_Screen_Lock_Computer_Settings.PNG)

### User Configuration
**Path:** User Configuration → Policies → Administrative Templates → Control Panel → Personalization

- **Enable screen saver** — Enabled
- **Screen saver timeout** — 300 seconds

![description](https://github.com/nullmarch/active-directory-lab/blob/main/Screenshots/GPO_Screen_Lock_User_Settings.PNG)

**Difference between Computer and User Configuration:**
Computer Configuration applies to the machine regardless of who logs in. User Configuration applies to the user account regardless of which machine they use. Both were configured for defense in depth — the machine inactivity limit provides a hard OS-level lock while the screen saver settings enforce the visual lock at the user level.

---

## 4. GPO 2 — Wallpaper Policy

A shared folder was created on the domain controller containing a wallpaper image, then deployed to all domain computers via GPO.

![description](https://github.com/nullmarch/active-directory-lab/blob/main/Screenshots/Wallpaper_Policy_GPO.PNG)


### Shared Folder Setup

The folder was shared with two permission layers:

**Network Share Permission** (Sharing tab):
- Authenticated Users — Read access
- Controls who can see and connect to the folder over the network

**NTFS Permission** (Security tab):
- Authenticated Users — Read & Execute
- Controls what users can do once connected

Both permissions must allow access. The most restrictive of the two applies. Network share permission grants visibility — NTFS permission grants actual read access to the files inside.

![description](https://github.com/nullmarch/active-directory-lab/blob/main/Screenshots/Shared_folder_NetSharePerm.PNG)

![description](https://github.com/nullmarch/active-directory-lab/blob/main/Screenshots/NTFS_Security_Tab.PNG)


### GPO Configuration
**Path:** User Configuration → Policies → Administrative Templates → Desktop → Desktop

- **Desktop Wallpaper** — set to network path `\\DC01\Wallpaper\duck.jpg`

![description](https://github.com/nullmarch/active-directory-lab/blob/main/Screenshots/GPO_Wallpaper_Network_Path_Settings.PNG)

![description](https://github.com/nullmarch/active-directory-lab/blob/main/Screenshots/duck-wallpaper.png)


## 5. Verification

After configuring both GPOs, the Windows 11 client ran:

```bash
gpupdate /force
````
to immediately pull the latest policies from the domain controller without waiting for the automatic 90-minute refresh cycle.
```bash
gpresult /r
````

confirmed both policies were applied under the correct user and computer sections.

![description](https://github.com/nullmarch/active-directory-lab/blob/main/Screenshots/policy-updates.png)



---

## What I Learned

### Active Directory Structure
Separating user and computer objects into dedicated OUs allows independent GPO targeting. A policy linked at the parent OU inherits down to all sub-OUs automatically.

### Group Policy
GPOs have two independent sections — Computer Configuration and User Configuration. Computer settings apply to the machine regardless of who logs in. User settings apply to the account regardless of which machine is used. Both can run simultaneously for layered enforcement.

### Shared Folder Permissions
Two permission layers control network folder access:
- **Network share permissions** — control visibility and connection to the folder
- **NTFS permissions** — control file level access rights once connected

Both must permit access. The most restrictive of the two takes effect.

### Domain DNS Dependency
Active Directory requires DNS to function. The domain controller acts as the DNS server. Client machines must point to the domain controller IP as their DNS server to locate and authenticate with the domain.

### GPO Verification
- `gpupdate /force` — triggers immediate policy refresh without waiting for the automatic 90 minute cycle
- `gpresult /r` — confirms which policies applied and to which scope, useful for troubleshooting inheritance issues

### Defense in Depth
Applying both machine inactivity limit and screen saver timeout creates layered security — one enforced at OS level, one at user level. Neither alone is as strong as both together.
