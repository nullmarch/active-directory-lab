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


# GPO 3 — Password Policy and Account Lockout Policy

## Why These Policies

Password and account lockout policies are foundational security controls in any enterprise environment. They directly address two of the most common attack vectors:

- **Weak passwords** — easily guessed or cracked through brute force or dictionary attacks
- **Unrestricted login attempts** — allows attackers to attempt unlimited password guesses without consequence

The values configured in this lab reflect industry standard baselines used in regulated environments including healthcare and pharmaceutical companies:

- **8 character minimum** — NIST and CIS benchmarks recommend 8 as the absolute minimum for domain accounts
- **Complexity requirements** — forces a mix of uppercase, lowercase, numbers, and symbols — significantly increases password search space
- **90 day maximum age** — limits the window an attacker can use a compromised credential
- **1 day minimum age** — prevents users from immediately cycling back to a previous password
- **5 password history** — prevents reuse of recent passwords
- **5 failed attempt lockout** — stops brute force attempts cold after 5 failures
- **15 minute lockout duration** — long enough to deter attackers, short enough to avoid excessive helpdesk load

---

## Initial Configuration

Two separate domain-level GPOs were created initially:

- **Security Policy** — containing password policy settings
- **Account Lockout Policy** — containing lockout settings

Both were linked at the ducklab.local domain root level.

![Pre-solution GPOs linked at domain root](https://github.com/nullmarch/active-directory-lab/blob/main/Screenshots/gpo3/Password_Policy_GPO.PNG)

![Pre-solution GPOs linked at domain root](https://github.com/nullmarch/active-directory-lab/blob/main/Screenshots/gpo3/GPO_Account_Lockout_Policy.PNG)

**Security Policy configuration:**

- Minimum password length: 8 characters
- Password complexity requirements: Enabled
- Maximum password age: 90 days
- Minimum password age: 1 day
- Enforce password history: 5 passwords

**Account Lockout Policy configuration:**

- Account lockout threshold: 5 failed logon attempts
- Account lockout duration: 15 minutes
- Reset account lockout counter after: 15 minutes

![Security Policy GPO settings](https://github.com/nullmarch/active-directory-lab/blob/main/Screenshots/gpo3/Password_Policy_config.PNG)

![Account Lockout GPO settings](https://github.com/nullmarch/active-directory-lab/blob/main/Screenshots/gpo3/Account_Lockout_Policy_config.PNG)

---

## Troubleshooting — Policies Not Applying

After configuration, verification from the Windows 11 client showed the expected settings were not appearing.

**Initial domain connectivity checks performed:**

```bash
whoami
nltest /dsgetdc:ducklab.local
nslookup
ping
```

All confirmed the client was communicating with the domain controller correctly. The issue was not domain connectivity.

### Group Policy Results Wizard — RPC Failure

Attempted to generate a Group Policy Results report remotely from the domain controller. The wizard failed with:

- RPC server unavailable
- WMI communication issue

This indicated Windows Firewall on the Windows 11 client was blocking remote management communication. The following inbound firewall rules were enabled under the Domain profile on Windows 11:

- Windows Management Instrumentation (WMI-In)
- Remote Service Management
- Remote Event Log Management
- File and Printer Sharing

After enabling these rules the Group Policy Results Wizard connected successfully.

### Finding the Root Cause

The generated report revealed that the newly created GPOs were not controlling the effective account policy. The **Default Domain Policy** was taking precedence.

This is by design in Active Directory — password and account lockout policies are special domain-level settings. When multiple GPOs define the same account policy settings, precedence conflicts arise. The Default Domain Policy, being the oldest and highest priority GPO at the domain level, won over the newly created GPOs.

![GPO Results Wizard showing Default Domain Policy winning](https://github.com/nullmarch/active-directory-lab/blob/main/Screenshots/gpo3/GPRW_Report.PNG)

---

## Resolution

Password policy and account lockout settings were moved into the **Default Domain Policy** directly — the correct and intended location for these settings in Active Directory.

The separate Security Policy and Account Lockout Policy GPOs were unlinked from the domain to prevent precedence conflicts with the Default Domain Policy.

Other policies remained in their correct locations:

- Screen Lock Policy — linked to Lab Users OU
- Wallpaper Policy — linked to Lab Users OU

![Default Domain Policy — Password Policy settings applied](https://github.com/nullmarch/active-directory-lab/blob/main/Screenshots/gpo3/GPO_DefDomPolicy_PassPolic.PNG)

![Default Domain Policy — Account Lockout settings applied](https://github.com/nullmarch/active-directory-lab/blob/main/Screenshots/gpo3/GPO_DefDomPolicy_AccLockPolic.PNG)

---

## Verification

A new Group Policy Results report confirmed the effective settings were now controlled by Default Domain Policy.

![GPO Results Wizard showing correct effective policy](https://github.com/nullmarch/active-directory-lab/blob/main/Screenshots/gpo3/new_GPRW_REPORT.PNG)

Verified effective policy from the Windows 11 client using:

```bash
net accounts
```

Output confirmed all configured values were applied correctly.

![net accounts output on Windows 11](https://github.com/nullmarch/active-directory-lab/blob/main/Screenshots/gpo3/confirm_policy_jsmith.png)

> **Important — gpresult user context:**
> Running `gpresult /r` as Administrator shows the Administrator account's applied policy — not the target user's. To check jsmith's effective policy, run gpresult while logged in as jsmith, or use:
> ```bash
> gpresult /r /user ducklab\jsmith
> ```

---

## Testing

### Password Policy Test

Attempted to change jsmith's password to a weak password that did not meet complexity requirements.

**Result:** Windows rejected the password and displayed complexity requirements.

Created a compliant password meeting all requirements.

**Result:** Password change succeeded.

![Windows rejecting weak password on login screen](https://github.com/nullmarch/active-directory-lab/blob/main/Screenshots/gpo3/confirm_password_policy_enforcement.png)

### Account Lockout Test

Logged in as jsmith and intentionally entered an incorrect password more than 5 times.

**Result:** Windows delayed further authentication attempts and the account became locked after reaching the threshold.

Confirmed the lockout in two locations:

- Windows 11 login screen showing account locked message
- Active Directory Users and Computers on the server showing the account as locked in jsmith's user properties

![Windows 11 login screen showing account locked](https://github.com/nullmarch/active-directory-lab/blob/main/Screenshots/gpo3/account_locked.png)

![jsmith account locked in ADUC on server](https://github.com/nullmarch/active-directory-lab/blob/main/Screenshots/gpo3/confirm_account_lock_AD.PNG)

---

## What I Learned

**GPO precedence and account policies:**
Password and account lockout policies behave differently from standard GPO settings. They must be configured in the Default Domain Policy at the domain root level. Multiple GPOs defining the same account policy settings cause precedence conflicts — the Default Domain Policy will win.

**Effective policy vs configured policy:**
A GPO can be configured correctly but still not apply if another GPO takes precedence. Always verify the effective policy using Group Policy Results or `net accounts` — not just the settings inside the GPO editor.

**Group Policy Results Wizard:**
Essential for troubleshooting GPO application. Shows exactly which GPO is controlling each setting and why others lost.

**RPC and WMI for remote management:**
Windows Firewall blocks remote administrative tools even when domain connectivity works. RPC and WMI inbound rules must be enabled on the client for remote Group Policy reporting to function.

**gpresult user context:**
Running gpresult as a different user shows that user's policy, not the target account's. Always run gpresult in the correct user context or specify the target user explicitly.

**Default Domain Policy:**
Should not be heavily modified for general settings — it exists specifically for domain-wide account policies. Other policies should be separated into dedicated GPOs linked at the appropriate OU level.
