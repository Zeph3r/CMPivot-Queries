# CMPivot Query Reference

A working set of CMPivot queries for common operational and compliance checks.

Two things apply throughout:

- **Online only.** CMPivot returns data only from clients that are online at query time — it is not a database query. This especially affects the stale-client and missing-software checks below.
- **Version strings compare lexically.** `ProductVersion` and build values are compared as text, so `'9.0.10' < '9.0.9'` evaluates `true`. Match exact known values when a comparison crosses a digit boundary.

## Find Devices with Pending Reboots
Surfaces devices with a restart waiting. The ConfigMgr client's own reboot
coordinator key only holds values when a reboot is actually pending, so any
device that returns a `RebootBy` row has a CM-coordinated reboot queued.
**An empty key means no pending reboot.**
```kusto
// ConfigMgr client reboot coordinator (most relevant for patch deployments)
Registry('HKLM:\SOFTWARE\Microsoft\SMS\Mobile Client\Reboot Management\RebootData')
| where Property == 'RebootBy'
| project Device, Property, Value

// Windows-side sources — a row is returned only if the key exists:
// RegistryKey('HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Component Based Servicing\RebootPending')
// RegistryKey('HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\WindowsUpdate\Auto Update\RebootRequired')
```

## Validate Application Deployment / Version
Confirms a deployment landed at the expected version across the collection by
showing the spread of installed versions. **`like` is unreliable against
`ProductVersion` — use `==` or `>`/`<` there; it works fine on `ProductName`.**
```kusto
InstalledSoftware
| where ProductName like 'Google Chrome'
| summarize dcount(Device) by ProductName, ProductVersion
```

## Detect Microsoft Office Installations (Pre-OffScrub Targeting)
Identifies devices with Microsoft Office components using SCCM
InstalledSoftware inventory. CMPivot does not reliably distinguish
MSI-based Office from Click-to-Run; therefore this query is intended
for **initial scoping only**.
```kusto
InstalledSoftware
| where Publisher == 'Microsoft Corporation'
| where ProductName contains 'Office'
| project Device, ProductName
```

## Identify Devices Missing Required Software
Lists devices that lack a required product. A device missing the target still
returns its other installed-software rows, so it appears in the per-device
summary with a count of 0. **A powered-off device won't appear at all — re-run
as stragglers come online.**
```kusto
InstalledSoftware
| summarize countif(ProductName == 'CrowdStrike Windows Sensor') by Device
| where countif_ == 0
| project Device
```
A more robust variant anchors a left-outer join to a product known to be on
every machine (the ConfigMgr client), so nothing is missed:
```kusto
InstalledSoftware
| where ProductName like 'Configuration Manager Client'
| join kind=leftouter (InstalledSoftware | where ProductName like 'CrowdStrike Windows Sensor')
| where isnull(ProductName1)
| project Device
```

## Scope Vulnerable Software Versions
Scopes endpoints running a vulnerable build of a product. **CMPivot compares
`ProductVersion` as a string, so `'9.0.10' < '9.0.9'` is `true` — when the
comparison crosses a digit boundary, match the exact known-vulnerable versions
explicitly (e.g. `ProductVersion in ('114.0','114.0.1')`) instead of using `<`.**
```kusto
InstalledSoftware
| where ProductName like 'Mozilla Firefox' and ProductVersion < '115.0.1'
| project Device, ProductName, ProductVersion
```

## Find Stale SCCM Clients
A genuinely stale client is offline and **won't answer a CMPivot query at all** —
for those, use Monitoring > Client Status, the "Last Active Time" column under
Devices, or a WQL collection on `SMS_R_System`. What CMPivot *can* catch is
online machines whose client agent is broken:
```kusto
Service
| where Name == 'CcmExec' and State != 'Running'
| project Device, Name, State, StartMode
```

## Detect Failed Services
Finds services that are supposed to be running but aren't. **Filtering on
`StartMode == 'Auto'` keeps legitimately-stopped manual and disabled services
out of the results.**
```kusto
Service
| where StartMode == 'Auto' and State != 'Running'
| project Device, Name, DisplayName, State, StartMode, Status
```

## Validate Windows Build Compliance
Lists devices not on the approved OS build. `OS` returns one row per device, so
filtering on `Version` cleanly flags every non-compliant endpoint.
```kusto
OS
| where Version != '10.0.19045'
| project Device, Caption, Version, BuildNumber
```

## Validate Windows 11 via NT Build Number
Identifies Windows 11 endpoints using the authoritative NT build number.
This avoids Microsoft's backward-compatibility behavior where Windows 11
may still report as "Windows 10" internally in SCCM / CMPivot.
**Windows 11 NT builds start at 22000 and above.**
```kusto
Registry('HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion')
| where Property == 'CurrentBuild'
| extend BuildNumber = toint(Value)
| project Device, BuildNumber, IsWindows11 = BuildNumber >= 22000
```

## Identify Low Disk Space Before Deployment
Flags endpoints below a free-space threshold ahead of a deployment.
`FreeSpace` and `Size` come from `Win32_LogicalDisk` in **bytes**, hence the
divide-by-1024³. Drop the `DeviceID == 'C:'` filter to evaluate all fixed drives.
```kusto
Disk
| where Description == 'Local Fixed Disk' and DeviceID == 'C:'
| extend FreeGB = FreeSpace / 1073741824.0
| where FreeGB < 10
| project Device, DeviceID, FreeGB, Size
| order by FreeGB asc
```

## Verify a Remediation Changed a Registry Value
Confirms a remediation wrote the expected value. **Values return as strings —
compare against `'0'`/`'1'`, not bare numbers.** Add `| where Value != '0'` to
show only the machines where the change didn't take.
```kusto
Registry('HKLM:\SOFTWARE\Policies\Microsoft\Windows\WinRM\Service')
| where Property == 'AllowUnencryptedTraffic'
| project Device, Property, Value
```

## Check Certificate Expiration (Not Native to CMPivot)
**CMPivot has no certificate entity** — it can't enumerate the certificate store
or read expiry dates, and registry-stored certificates come back as unparseable
binary blobs. Use a ConfigMgr **Run Script** (Software Library > Scripts) instead,
which runs against a collection on demand much like CMPivot:
```powershell
Get-ChildItem Cert:\LocalMachine\My |
  Where-Object { $_.NotAfter -lt (Get-Date).AddDays(30) } |
  Select-Object Subject, Thumbprint, NotAfter
```
