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
