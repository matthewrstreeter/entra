# Conditional Access Policy Export and GUID Resolution

[`CA-ExportPolices_ResolveGuids.ps1`](CA-ExportPolices_ResolveGuids.ps1) exports Microsoft Entra Conditional Access policies, resolves referenced directory object IDs where possible, and creates JSON and CSV reports. The original policy JSON exports are kept separate and are not modified.

## Prerequisites

- PowerShell with the Microsoft Graph PowerShell SDK modules listed below.
- A Microsoft Entra account permitted to read Conditional Access policies and directory objects.
- Delegated Microsoft Graph permissions: `Policy.Read.All`, `Directory.Read.All`, and `Application.Read.All`. Your organization may require an administrator to grant consent.

Install the required modules for your user:

```powershell
Install-Module -Name Microsoft.Graph.Authentication -Scope CurrentUser
Install-Module -Name Microsoft.Graph.Identity.SignIns -Scope CurrentUser
```

## Usage

From the repository root, run:

```powershell
.\Entra\CA-ExportPolices_ResolveGuids.ps1
```

By default, output is written to `.\Temp\CAPolicies` relative to the current working directory. Choose another output folder with `-OutputFolder`:

```powershell
.\Entra\CA-ExportPolices_ResolveGuids.ps1 -OutputFolder "C:\Temp\CAPolicies"
```

The script connects to Microsoft Graph with the required scopes if there is no existing Graph session. If a session already exists, it reuses it; ensure that session has the required delegated permissions. To reconnect with the requested scopes:

```powershell
Disconnect-MgGraph
Connect-MgGraph -Scopes "Policy.Read.All", "Directory.Read.All", "Application.Read.All"
```

## Output

The output folder contains:

| Path | Contents |
| --- | --- |
| `Original/` | One JSON file per exported policy, retaining the original object IDs. |
| `Resolved/` | A JSON copy of each processed policy with referenced values replaced by resolution objects. |
| `ConditionalAccessPolicy-ResolvedAssignments.csv` | Detailed rows for resolved assignments, including policy, category, include/exclude assignment, original value, resolved name, object ID, and status. |
| `ConditionalAccessPolicy-PolicySummary.csv` | One summary row per processed policy, including assignment names, policy conditions, grant controls, and object counts. |

The script attempts to resolve users, groups, applications/cloud apps, service principals, directory roles, and named locations. Built-in values such as `All` are labeled as built-in rather than looked up. Values that cannot be resolved are retained with a `PossiblyDeletedOrUnavailable` status and a resolution message; this does not necessarily mean the object was deleted.

## Notes

- The output folder is reused and is not cleared before a run. Existing JSON files in `Original/` are also processed, so move or remove stale exports before rerunning if you need a clean report.
- Application assignments generally contain an application ID (client ID); client application service-principal assignments contain service-principal object IDs.
- Treat the exports as sensitive administrative data and store them accordingly.