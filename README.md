# Windows Autopilot Pre-Production Deployment

## About this project

I built this Windows Autopilot pre-production environment to design, configure and validate a modern Windows provisioning process using Microsoft Intune and Microsoft Entra ID.

The purpose was to work through the deployment from beginning to end in a controlled environment before using the same approach for a wider enterprise rollout. I wanted to confirm not only that Autopilot enrollment worked, but also that the device received the expected Intune configuration and security controls after provisioning.

For the endpoint, I used a Windows 11 Pro virtual machine in VMware Workstation. This allowed me to repeat OOBE, make configuration changes, troubleshoot issues and validate the deployment without affecting existing devices.

The work covered Autopilot registration, dynamic device targeting, a user-driven deployment profile, Microsoft Entra Join, automatic Intune enrollment, Enrollment Status Page (ESP), compliance and configuration policies, BitLocker and Windows LAPS.

**Pre-production environment:** Windows 11 Pro | VMware Workstation | Microsoft Intune | Microsoft Entra ID

---

## Deployment workflow

<img width="1024" height="1536" alt="Windows Autopilot GitHub" src="https://github.com/user-attachments/assets/6f731f05-f185-4f77-b56a-a24ae0447205" />



The provisioning path I configured and validated was:

**Windows OOBE → Windows Autopilot → Microsoft Entra Join → Automatic Intune Enrollment → ESP → Compliance & Configuration Policies → BitLocker + Windows LAPS**

---

## What I configured

- Windows Autopilot device registration
- Dynamic Microsoft Entra device group
- User-driven Windows Autopilot deployment profile
- Microsoft Entra Join
- Automatic Microsoft Intune enrollment
- Enrollment Status Page (ESP)
- Windows configuration policy
- Windows compliance policy
- BitLocker disk encryption policy
- Windows LAPS with password backup to Microsoft Entra ID
- End-to-end testing and troubleshooting

---

## 1. Pre-production Windows 11 device

I created a Windows 11 Pro VM in VMware Workstation and used it as the pre-production endpoint.

The VM was configured with TPM support and the required Windows 11 resources. Keeping the pre-production device separate also meant I could reset Windows and repeat the OOBE/Autopilot process whenever I needed to test a configuration change.

---

## 2. Autopilot device registration

The first step was registering the Windows device with Autopilot.

During Windows OOBE, I opened PowerShell and collected the hardware information required for Autopilot registration.

```powershell
Set-ExecutionPolicy -ExecutionPolicy Bypass -Scope Process -Force

Get-WindowsAutopilotInfo -OutputFile C:\AutopilotHWID.csv
```

I verified that the CSV had been created and copied it from the VM so it could be imported into Intune.

After importing the hardware CSV into Windows Autopilot, I confirmed that the VM appeared in the Autopilot device list.

This gave Intune the hardware identity it needed to recognise the device when it connected to the Autopilot service during OOBE.

---

## 3. Dynamic Autopilot device group

Rather than manually adding Autopilot devices to a deployment group, I created a dynamic Microsoft Entra device group.

The membership rule was:

```text
(device.devicePhysicalIDs -any (_ -startsWith "[ZTDid]"))
```

This identifies registered Windows Autopilot devices using the Autopilot device identifier.

After importing the VM, I confirmed that it automatically became a member of my pre-production Autopilot device group.

I then used this group as the deployment scope for the Autopilot configuration.

---

## 4. Autopilot deployment profile

I created a user-driven Windows Autopilot deployment profile configured for **Microsoft Entra Join**.

The profile controlled the Windows OOBE experience and included settings for:

- User-driven deployment
- Microsoft Entra Join
- Privacy settings
- Microsoft software licence terms
- Keyboard configuration
- Account options during OOBE

I assigned the profile to the pre-production Autopilot device group and waited until the device showed the profile as assigned before starting the deployment test.

This was an important check because I wanted to make sure the device had received its Autopilot profile before resetting it back to OOBE.

---

## 5. Enrollment Status Page

I configured an Enrollment Status Page (ESP) as part of the provisioning process.

I wanted the device to go through a managed provisioning experience while Intune processed the required policies and configurations rather than simply completing enrollment and checking the device afterwards.

ESP also gave me a clearer view of the provisioning stage during the Autopilot deployment.

---

## 6. Microsoft Entra Join and automatic Intune enrollment

Once the Autopilot profile was assigned, I returned the Windows VM to OOBE and started the deployment again.

The registered hardware was recognised by Windows Autopilot and the assigned deployment profile was applied.

After signing in with a test organisational account, the device:

- joined Microsoft Entra ID
- automatically enrolled into Microsoft Intune
- received the policies assigned to the pre-production device scope

I checked the device join state locally with:

```powershell
dsregcmd /status
```

The main values I wanted to confirm were:

```text
AzureAdJoined : YES
DomainJoined : NO
DeviceAuthStatus : SUCCESS
```

This confirmed that the endpoint was Microsoft Entra joined and that this was a cloud-native deployment rather than an on-premises Active Directory domain join.

---

## 7. Intune compliance and configuration

Provisioning the device was only part of the validation.

I also wanted to confirm that the endpoint moved into the expected management state after Autopilot completed.

I created and assigned Windows compliance and configuration policies to the pre-production device scope.

After enrollment, I checked the endpoint in Intune and confirmed that it was:

- managed by Microsoft Intune
- recognised as a corporate device
- reporting as compliant

This confirmed that the device had moved successfully from Autopilot provisioning into ongoing Intune management.

---

## 8. BitLocker management

I configured BitLocker through Microsoft Intune Endpoint Security.

Rather than enabling encryption manually on the endpoint, I wanted BitLocker to be part of the centrally managed security configuration applied to the provisioned device.

The BitLocker policy was assigned through Intune and used to validate disk-encryption management and recovery-key handling for the Windows endpoint.

This gave me an end-to-end path where the device was provisioned through Autopilot and then received its encryption requirements through Intune.

---

## 9. Windows LAPS

I also added Windows Local Administrator Password Solution (LAPS) to the pre-production configuration.

My objective was to avoid a static local Administrator password and instead have Windows LAPS manage and rotate the password automatically.

I configured:

- Windows LAPS policy through Intune
- Microsoft Entra LAPS
- local Administrator password rotation
- password backup to Microsoft Entra ID
- password recovery through Microsoft Entra ID

This meant the local Administrator account on the managed endpoint could have its own automatically rotated password rather than relying on a common local administrator credential.

---

## LAPS troubleshooting

The LAPS configuration did not work correctly on the first attempt.

Instead of changing the policy repeatedly, I checked the Windows LAPS operational event log on the endpoint to see what was actually happening.

I manually triggered LAPS policy processing with:

```powershell
Invoke-LapsPolicyProcessing
```

I then reviewed the LAPS operational events:

```powershell
Get-WinEvent -LogName "Microsoft-Windows-LAPS/Operational" -MaxEvents 10 |
Select-Object TimeCreated,Id,LevelDisplayName,Message |
Format-List
```

The logs showed that the policy was being processed but the password could not be backed up to Microsoft Entra ID.

The error led me back to the tenant configuration, where I found that Microsoft Entra LAPS had not been enabled.

After enabling **Microsoft Entra Local Administrator Password Solution (LAPS)** at tenant level, I triggered policy processing again.

The later LAPS events confirmed that:

- policy processing completed successfully
- the local Administrator password was updated
- the new password was successfully backed up to Microsoft Entra ID

I then verified the device through the Local Administrator Password Recovery area in Microsoft Entra ID.

This was a useful troubleshooting exercise because the Intune policy itself was not the problem. The endpoint logs pointed to a tenant-side dependency that still needed to be enabled.

---

## Autopilot connectivity troubleshooting

During Autopilot testing, I also checked whether the Windows endpoint could reach the Microsoft Autopilot service.

I used:

```powershell
Test-NetConnection ztd.dds.microsoft.com -Port 443
```

The result showed:

```text
TcpTestSucceeded : True
```

This confirmed successful TCP 443 connectivity from the endpoint to the Autopilot service.

I also checked the local Autopilot diagnostic information:

```powershell
reg Query HKLM\SOFTWARE\Microsoft\Provisioning\Diagnostics\Autopilot /s
```

These checks helped separate network/service connectivity from profile or enrollment configuration when troubleshooting the deployment.

---

## Validation result

After completing the configuration and troubleshooting, I repeated the provisioning process and validated the device from both the Windows endpoint and the Microsoft management portals.

### ✅ Pre-Production Validation Completed

| Validation | Result |
|---|---|
| Windows Autopilot profile | ✅ Applied |
| Microsoft Entra Join | ✅ Successful |
| Automatic Intune enrollment | ✅ Successful |
| Corporate device management | ✅ Confirmed |
| Device compliance | ✅ Compliant |
| Configuration policies | ✅ Applied |
| BitLocker management | ✅ Validated |
| Windows LAPS | ✅ Password rotation and Entra backup validated |

The pre-production endpoint successfully completed the full provisioning and management workflow:

**Windows Autopilot → Microsoft Entra Join → Intune Enrollment → ESP → Compliance & Configuration → BitLocker + Windows LAPS**

This provided a validated baseline for the Autopilot provisioning and endpoint-management configuration before a broader enterprise rollout.

---

## Key commands used

### Collect Autopilot hardware information

```powershell
Set-ExecutionPolicy -ExecutionPolicy Bypass -Scope Process -Force

Get-WindowsAutopilotInfo -OutputFile C:\AutopilotHWID.csv
```

### Check Microsoft Entra Join status

```powershell
dsregcmd /status
```

### Test Autopilot connectivity

```powershell
Test-NetConnection ztd.dds.microsoft.com -Port 443
```

### Check Autopilot diagnostic information

```powershell
reg Query HKLM\SOFTWARE\Microsoft\Provisioning\Diagnostics\Autopilot /s
```

### Trigger Windows LAPS processing

```powershell
Invoke-LapsPolicyProcessing
```

### Review Windows LAPS events

```powershell
Get-WinEvent -LogName "Microsoft-Windows-LAPS/Operational" -MaxEvents 10 |
Select-Object TimeCreated,Id,LevelDisplayName,Message |
Format-List
```

---

## Skills covered in this project

### Windows endpoint management

- Windows Autopilot
- Microsoft Intune
- User-driven provisioning
- Enrollment Status Page
- Device compliance
- Configuration profiles
- Corporate device management

### Identity and device registration

- Microsoft Entra ID
- Microsoft Entra Join
- Dynamic device groups
- Autopilot hardware registration
- Cloud-based Windows device identity

### Endpoint security

- BitLocker
- Windows LAPS
- Microsoft Entra LAPS
- Local Administrator password rotation
- Recovery-key and credential management
- Intune Endpoint Security

### Troubleshooting and validation

- PowerShell
- `dsregcmd`
- Windows Event Viewer
- Windows LAPS operational logs
- Autopilot diagnostic registry information
- Network connectivity testing
- Intune and Microsoft Entra portal validation

---

## Full project documentation

I documented the full implementation separately, including the configuration steps, screenshots, PowerShell commands, troubleshooting and validation evidence.

**[View the Windows Autopilot Pre-Production Deployment PDF](./Windows%20Autopilot%20Pre-Production%20Deployment.pdf)**

---

## Security and privacy

The public documentation in this repository has been sanitized before publication.

Tenant-specific information, usernames, email addresses, device identifiers, serial numbers, recovery credentials, passwords and other environment-specific identifiers visible in the original implementation have been removed or redacted from the published screenshots.

No production credentials or confidential organisational information are included.
