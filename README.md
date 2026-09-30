# Why Windows LAPS Only Showed the Default Admin on Some of Our New Devices

*Windows LAPS · Intune · Entra ID · Windows Autopatch*

We rolled out Windows LAPS through Intune with **automatic account management**, where Windows creates its own dedicated local admin account and rotates its password. On most devices it worked as expected. On a few, Entra ID only ever showed the **default built-in Administrator**, and the custom account was never created.

The fix turned out to be simple. Finding the cause took a lot longer.

---

## The symptom

A handful of devices behaved differently from the rest:

- Intune reported the LAPS policy as **Succeeded**
- A password was backed up to Entra ID/Intune and was rotating
- But the managed account didn't exist, and Entra ID/Intune listed only **Administrator**

The confusing part was that these devices came from **the same batch as devices that worked**. Same hardware, same policy, same assignment. With a few failures scattered among working devices, nothing obviously pointed to a root cause. With other projects running at the same time, I didn't get a chance to look at it properly.

<img width="512" height="520" alt="default Administrator" src="https://github.com/user-attachments/assets/d4227fb0-0865-4bde-b50f-9e12cfa85e58" />

*Intune showing only the default Administrator*

---

## How I found it

The answer came from somewhere I wasn't looking. We're in the middle of moving our devices to **Windows Autopatch**. When I finally had time to go through the tenant and review the new Autopatch and update reports, one of them showed that **quite a few devices were running Windows versions that were out of support, or soon would be.**

When I looked at which devices those were, I recognised them straight away from our **new naming convention**. They were brand-new devices.

They were on **Windows 11 21H2**. The vendor had shipped them with a build that was already past end of servicing, and I hadn't expected that. I work at L3 now, so I don't set up new devices myself the way I used to. Back then I'd have noticed immediately, because 21H2 looks noticeably different from 25H2.

That made me check whether these were the same devices with the LAPS problem. They were.

<img width="1620" height="919" alt="unsupported Windows versions" src="https://github.com/user-attachments/assets/08a53fe3-29bf-416b-882a-809fbf779706" />

<img width="1198" height="901" alt="Windows versions" src="https://github.com/user-attachments/assets/64e6ac6a-7455-4298-a853-8a952ad740f6" />

*Autopatch / update report showing devices on unsupported Windows versions*

---

## The root cause

Windows LAPS came to Windows 11 21H2 in the April 2023 update, but **automatic account management requires Windows 11 24H2 or later.** Older builds skip settings they don't support, **without any error**, and fall back to managing the built-in Administrator. That account is disabled by default.

The result was a rotated password for an account nobody could sign in with, while Intune reported that everything was fine.

| | Windows 11 21H2 | Windows 11 25H2 |
|---|---|---|
| LAPS backup to Entra ID | ✅ | ✅ |
| Automatic account management | ❌ Ignored | ✅ |
| Account shown in Entra ID | Built-in Administrator | Custom managed account |

---

## The fix

I upgraded the affected devices to **Windows 11 25H2** with an Intune feature update policy. After the next sync, the custom account was created and enabled, its password was backed up to Entra ID under the correct name, and it rotated after use.

<img width="514" height="486" alt="LAPS after update" src="https://github.com/user-attachments/assets/c3f057d5-0346-41c0-b748-3e7bd30a96ab" />

*Custom managed account visible in Entra ID after the upgrade*

---

## The setup, briefly

**Entra ID:** Device settings → *Enable Microsoft Entra LAPS* = Yes

**Intune:** Endpoint security → Account protection → Windows LAPS

| Setting | Value |
|---|---|
| Backup Directory | Microsoft Entra ID only |
| Password Age Days | 7 |
| Post Authentication Actions | Reset password and log off |
| Automatic Account Management | Enabled, new custom account *XXXXX* |
| Account Name / Prefix | e.g. `XXXXX` |

**Quick checks on a device:**

```powershell
Invoke-LapsPolicyProcessing
Get-WinEvent -LogName "Microsoft-Windows-LAPS/Operational" -MaxEvents 10
Get-LocalUser
```
<img width="1099" height="613" alt="PS check" src="https://github.com/user-attachments/assets/ae901a16-2584-48e6-84fb-22f4374f8fdd" />

---

## Lessons learned

- **"Succeeded" in Intune doesn't mean every setting was applied.** Unsupported settings are skipped without an error.
- **If only some devices in a batch fail, compare OS builds first.**
- **Don't assume new hardware arrives on a current OS build.** Check it, or enforce a minimum OS version in a compliance policy.
- **Reporting catches what you miss.** The Autopatch and update reports found in minutes something I'd been working around for weeks.
