# Entra ID Authentication Method Reports

This directory contains PowerShell scripts for reporting authentication method registration status in Microsoft Entra ID.

## AuthMethods-SecurityQuestionsRegistration-GroupAuto.ps1

`AuthMethods-SecurityQuestionsRegistration-GroupAuto.ps1` resolves an Entra ID group, retrieves its direct user members, and checks whether those users have registered **Security Questions** as an authentication method. It exports a membership CSV, a detailed registration report, and a store-level follow-up report. Optional LDAPS enrichment adds on-premises Active Directory password-expiration information.

## Prerequisites

- PowerShell 7.0 or later
- Microsoft Graph PowerShell Authentication module (`Microsoft.Graph.Authentication`)
- Delegated Microsoft Graph permissions:
  - `Group.Read.All`
  - `User.Read.All`
  - `AuditLog.Read.All`
- An Entra role that permits reading authentication method registration reports, such as **Reports Reader**
- An output directory path where the script can create reports

The script imports `Microsoft.Graph.Authentication` and connects to Microsoft Graph if there is no existing Graph session. If a delegated session already exists, it verifies that the required scopes are present. An administrator may need to grant consent for the delegated permissions.

For optional password-expiration enrichment, see [LDAPS Password Enrichment](#ldaps-password-enrichment).

## Parameters

Run the script with parameters to select the group and customize processing:

| Parameter | Default | Description |
| --- | --- | --- |
| `-GroupName` | `GROUPNAME` | Display name of the Entra group. The script requires exactly one group with this name. |
| `-GroupId` | — | Group object ID. When specified, the script resolves the group directly by ID instead of by display name. |
| `-OutputDirectory` | `\Path\To\EntraReports` | Directory where the CSV reports are written. The script creates it if it does not exist. |
| `-MemberPageSize` | `999` | Number of group members requested per Microsoft Graph page (range: 1–999). |
| `-ReportPageSize` | `1000` | Number of registration records requested per Microsoft Graph page (range: 1–1000). |
| `-AccountStatus` | `All` | Users to include in the registration and store-level reports: `All`, `EnabledOnly`, or `DisabledOnly`. |
| `-SkipMembershipExport` | Off | Skip writing the group membership CSV. Membership is still retrieved and processed. |
| `-IncludePasswordExpiration` | Off | Add optional Active Directory password information through LDAPS. |
| `-DomainController` | — | Domain controller FQDN; required with `-IncludePasswordExpiration`. |
| `-LdapPort` | `636` | LDAPS port. |
| `-LdapPageSize` | `500` | Number of Active Directory entries requested per LDAP page (range: 100–1000). |
| `-LdapSearchBases` | Two example OUs | One or more LDAP search bases to search. Set these to the appropriate OUs in your environment when using LDAPS enrichment. |
| `-AdCredential` | — | Active Directory credential to use for LDAPS. |
| `-AdCredentialMode` | `Auto` | Credential retrieval mode: `Auto`, `Stored`, or `Interactive`. |
| `-AdSecretName` | `Company-AD-LDAPS` | Secret name used with SecretManagement in `Auto` or `Stored` mode. |
| `-AdSecretVault` | `LocalVault` | SecretManagement vault used in `Auto` or `Stored` mode. |

The checked-in defaults `GROUPNAME` and `\Path\To\EntraReports` are sanitized placeholders for publication, not environment-specific values. Replace them with your group's name and a valid output path in the script, or supply those values with `-GroupName` and `-OutputDirectory` when running it.

When using `-GroupName`, the script stops if no group or multiple groups have that display name; use `-GroupId` to select the intended group unambiguously.

## Usage

Run from PowerShell 7 or later. For example, to generate reports for a group by display name:

```powershell
.\AuthMethods-SecurityQuestionsRegistration-GroupAuto.ps1 `
    -GroupName "Example Group" `
    -OutputDirectory "C:\EntraReports"
```

To select a group by ID and skip the membership CSV:

```powershell
.\AuthMethods-SecurityQuestionsRegistration-GroupAuto.ps1 `
    -GroupId "00000000-0000-0000-0000-000000000000" `
    -OutputDirectory "C:\EntraReports" `
    -SkipMembershipExport
```

The script retrieves **direct user members** only; it does not expand nested groups.

## Account-Status Filtering

`-AccountStatus` filters which users are included in the registration and store-level reports:

- `All` includes enabled, disabled, and unknown-status members.
- `EnabledOnly` includes members whose `accountEnabled` value is true.
- `DisabledOnly` includes members whose `accountEnabled` value is false.

The membership CSV, when enabled, always includes all direct user members regardless of this filter. Unknown account status is included only with `-AccountStatus All`. If the filter selects no users, the script warns and does not generate the registration or store-level report.

## Output Files

The script creates filenames from the `-GroupName` value, replacing filename-invalid characters with underscores. When selecting a group with `-GroupId`, also supply `-GroupName` if you want the output filenames to use a meaningful group name; otherwise the default `GROUPNAME` is used. When `-AccountStatus` is `EnabledOnly` or `DisabledOnly`, the corresponding suffix is added to the registration and store report names. For example, with `-GroupName "Example Group"` and the default `All` filter:

- `Example Group-Members.csv`
- `Example Group-SecurityQuestionRegistration.csv`
- `Example Group-SecurityQuestionsByStore.csv`

The membership CSV is omitted when `-SkipMembershipExport` is used.

### Membership CSV

The membership CSV contains the group's direct user members with these fields:

| Column | Description |
| --- | --- |
| `Id` | Entra ID object ID |
| `userPrincipalName` | User principal name |
| `displayName` | Display name |
| `mail` | Email address |
| `employeeId` | Employee ID, used for store-level grouping |
| `AccountEnabled` | Account-enabled status (`True`, `False`, or blank if unknown) |
| `OnPremisesSecurityIdentifier` | On-premises security identifier, used for optional AD matching |

### Security Questions registration CSV

The registration CSV contains one row per user selected by `-AccountStatus`:

| Column | Description |
| --- | --- |
| `ObjectId` | Entra ID object ID |
| `DisplayName` | Display name |
| `UserPrincipalName` | UPN |
| `Mail` | Email address |
| `EmployeeId` | Employee ID |
| `AccountEnabled` | Account-enabled status |
| `SecurityQuestionsRegistered` | `True` or `False` when a registration record is found; blank when no record is found |
| `MethodsRegistered` | Semicolon-separated registered authentication methods; blank when no record is found |
| `RegistrationRecordFound` | Whether a registration record was matched by object ID |
| `PasswordLastSet` | AD password last-set timestamp in UTC, when available |
| `PasswordExpirationDate` | AD password-expiration timestamp in UTC, when available |
| `DaysUntilExpiration` | Days until expiration, when available |
| `PasswordNeverExpires` | Whether the AD account is configured so its password never expires, when available |
| `PasswordStatus` | AD password status; `Unknown` when enrichment is not enabled or no value is available |

Unmatched users are included with `RegistrationRecordFound` set to `False`. Their `SecurityQuestionsRegistered` field is blank, which distinguishes a missing registration record from a confirmed registration record without Security Questions.

### Store-level follow-up CSV

The store-level CSV groups the processed users by `EmployeeId`. Blank or whitespace-only employee IDs are grouped under `(Missing EmployeeId)`. It summarizes enabled, disabled, and unknown accounts; Security Questions registration and missing registration records; and enabled-user follow-up counts and registration percentages.

When LDAPS enrichment is enabled, it also includes counts of enabled users whose passwords are expired or expiring within 7, 14, or 30 days, plus counts with unknown password status. The 7-, 14-, and 30-day counts are cumulative and exclude already-expired passwords.

The report is sorted by enabled users needing follow-up, then enabled-user count, then employee ID. The console displays the top 15 groups; the CSV contains the full breakdown.

Store-level counts use the same filtered user population as the registration report. They do not necessarily represent every member when `-AccountStatus` is not `All`.

## LDAPS Password Enrichment

Password-expiration details are optional and are retrieved from on-premises Active Directory by matching each Entra user's `OnPremisesSecurityIdentifier` to an AD `objectSid`.

Example:

```powershell
.\AuthMethods-SecurityQuestionsRegistration-GroupAuto.ps1 `
    -GroupName "Example Group" `
    -OutputDirectory "C:\EntraReports" `
    -IncludePasswordExpiration `
    -DomainController "dc01.example.com" `
    -LdapSearchBases @(
        "OU=Users,DC=example,DC=com"
        "OU=Stores,DC=example,DC=com"
    )
```

Use search bases appropriate for your directory. The script connects over LDAPS using the configured port (636 by default) and searches each base and its subtrees. It reads password-related attributes and does not modify Active Directory.

With the default `Auto` credential mode, the script tries to retrieve a `PSCredential` from the configured SecretManagement vault and prompts interactively if retrieval fails. `Stored` mode requires the configured secret to be available and valid; `Interactive` mode prompts for credentials. You can also pass `-AdCredential` directly.

If LDAPS enrichment fails, the script warns and continues with the Graph registration report; password fields remain `Unknown` or blank. Password timestamps are reported in UTC.

## Console Summary and Notes

The console summary reports group membership and account-status counts, registration matches, Security Questions totals, and an enabled-user registration percentage. The percentage uses enabled users among the users processed by the selected filter; it is unavailable when using `-AccountStatus DisabledOnly`.

The script also prints progress and elapsed times for its main operations. Microsoft Graph report data may be subject to service-side availability or reporting delay. The script does not disconnect from Microsoft Graph automatically; run `Disconnect-MgGraph` afterward if needed.
