## Validate Windows 11 via NT Build Number

Identifies Windows 11 endpoints using the authoritative NT build number.
This avoids Microsoft’s backward-compatibility behavior where Windows 11
may still report as “Windows 10” internally in SCCM / CMPivot.

**Windows 11 NT builds start at 22000 and above.**

```kusto
Registry(
  'HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion',
  'CurrentBuild'
)
| summarize CurrentBuild = max(Value) by Device
| project Device, CurrentBuild, IsWindows11 = CurrentBuild >= '22000'
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
