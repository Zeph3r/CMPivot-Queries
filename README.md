# CMPivot-Queries

A small, curated collection of CMPivot queries for endpoint engineering and operational validation.

All queries are **detection-only** and do not modify endpoint state.

## Contents

- [Scope](#scope)
- [Requirements](#requirements)
- [Running a query](#running-a-query)
- [Query index](#query-index)
- [CMPivot limitations to know](#cmpivot-limitations-to-know)
- [Contributing](#contributing)
- [Inspiration](#inspiration)
- [License](#license)
- [Disclaimer](#disclaimer)

## Scope

This repository contains CMPivot queries intended for:

- Endpoint inventory and validation
- Configuration visibility
- Remediation scoping and verification

## Requirements

- Microsoft Configuration Manager (SCCM) current branch **1806 or later** — CMPivot was introduced in 1806.
- Queries that use the `RegistryKey` entity require **2107 or later**, and `RegistryKey` is **not supported on tenant-attached devices** via the Microsoft Intune admin center.
- Some entities require **PowerShell 5.0** on the target clients.
- An RBAC role with permission to run CMPivot against the target collection. The certificate workaround additionally uses the **Run Scripts** feature.
- Target devices must be **online** — see [limitations](#cmpivot-limitations-to-know).

## Running a query

In the Configuration Manager console:

1. Go to **Assets and Compliance > Device Collections**.
2. Right-click the target collection and choose **Start CMPivot**.
3. Paste a query into the query bar and run it.

CMPivot can also be run from the standalone CMPivot app, or from the Microsoft Intune admin center when tenant attach is enabled.

> **Tip:** Start against a small pilot collection. CMPivot fans out to every online client in the collection and can be bandwidth-heavy at scale.

## Query index

| Query | Purpose | Primary entity |
|---|---|---|
| Find Devices with Pending Reboots | Surface devices with a restart queued | `Registry` / `RegistryKey` |
| Validate Application Deployment / Version | Confirm a deployment landed at the expected version | `InstalledSoftware` |
| Detect Microsoft Office Installations | Initial scoping of Office components | `InstalledSoftware` |
| Identify Devices Missing Required Software | Find endpoints lacking a required product | `InstalledSoftware` |
| Scope Vulnerable Software Versions | Find endpoints on a vulnerable build | `InstalledSoftware` |
| Find Stale SCCM Clients | Catch online machines with a broken client agent | `Service` (+ Client Status) |
| Detect Failed Services | Find auto-start services that aren't running | `Service` |
| Validate Windows Build Compliance | Flag devices not on the approved OS build | `OS` |
| Validate Windows 11 via NT Build Number | Identify Windows 11 by authoritative NT build | `Registry` |
| Identify Low Disk Space Before Deployment | Flag endpoints below a free-space threshold | `Disk` |
| Verify a Remediation Changed a Registry Value | Confirm a remediation wrote the expected value | `Registry` |
| Check Certificate Expiration | Not native to CMPivot — uses a Run Script | `n/a` (Run Script) |

See [`cmpivot-query-reference.md`](./cmpivot-query-reference.md) for the full queries. *(Adjust the path/filename to match your repo layout.)*

## CMPivot limitations to know

These apply across the queries and are worth keeping in mind:

- **Online only.** CMPivot returns data only from clients online at query time — it is not a database query. This especially affects the stale-client and missing-software checks.
- **Version strings compare lexically.** `ProductVersion` and build values are compared as text, so `'9.0.10' < '9.0.9'` evaluates `true`. Match exact known values when a comparison crosses a digit boundary.
- **Operators are case-sensitive.** Use `where`, not `Where`.
- **String literals use single quotes.** For example, `where State == 'Stopped'`.
- **No HKCU.** Registry queries cannot target `HKEY_CURRENT_USER`.
- **No certificate entity.** CMPivot can't enumerate the certificate store; use a Run Script instead.
- **MSI vs Click-to-Run** Office installations are not reliably distinguished by `InstalledSoftware`; treat those queries as scoping only.

## Contributing

Contributions are welcome. To keep the collection consistent, please follow this format for each query:

````markdown
## Query Title

A one- or two-sentence description of what the query does, with **bold**
emphasis on any important caveat (online-only behavior, lexical comparisons, etc.).

```kusto
<query here>
```
````

Please test your query against a real collection before submitting, and note any version or entity requirements.

## Inspiration

Some of the initial structure and ideas for this repository were inspired by:
https://github.com/svschmit/CMPivot-Queries

If you're looking for a broader set of general CMPivot examples, that repo is a great resource.

## License

MIT
## Disclaimer

Provided as-is, with no warranty. Not affiliated with or endorsed by Microsoft. Validate against your own environment before broad use — entity availability and property names can vary across Configuration Manager builds.

*Validated against ConfigMgr current branch — update this line with the build you tested on.*
