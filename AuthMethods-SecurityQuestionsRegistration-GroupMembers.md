# Entra ID Authentication Method Reports

This directory contains PowerShell scripts for reporting authentication method registration status in Microsoft Entra ID.

## AuthMethods-SecurityQuestionsRegistration-GroupMembers.ps1

`AuthMethods-SecurityQuestionsRegistration-GroupMembers.ps1` checks whether members of an exported Entra ID group have registered **Security Questions** as an authentication method. It compares the group members' Entra object IDs with the Microsoft Graph authentication method registration details report, then writes a detailed CSV report.

## Prerequisites

- PowerShell 7 or Windows PowerShell
- Microsoft Graph PowerShell SDK with `Invoke-MgGraphRequest` available
- An authenticated Microsoft Graph PowerShell session with permission to read authentication method registration details
- An input CSV exported from Entra ID containing the required columns described below
- Existing output directories for the input and output files

This script does not connect to Microsoft Graph. Before running it, sign in with `Connect-MgGraph` and the permissions required to read the registration report. Depending on tenant policies, an administrator may need to grant consent.

## Input CSV

The input CSV should contain the following headers:

| Header | Required | Description |
| --- | --- | --- |
| `Id` | Yes | Entra ID object ID; used to match the user to the registration report |
| `userPrincipalName` | Yes | User principal name, included in the output report |
| `displayName` | Yes | Display name, included in the output report |
| `mail` | Yes | Email address, included in the output report |

Example:

```csv
Id,userPrincipalName,displayName,mail
00000000-0000-0000-0000-000000000001,alex@example.com,Alex Example,alex@example.com
00000000-0000-0000-0000-000000000002,jordan@example.com,Jordan Example,jordan@example.com
```

The script stops with an error if the CSV is empty or any required column is missing.

## Configuration

Open the script and update these variables before running it:

```powershell
$InputCsv = "\Path\To\Input\Members.csv"
$OutputCsv = "\Path\To\Output\Members-SecurityQuestionRegistration.csv"
$pageSize = 1000
```

`$pageSize` controls the number of registration records requested per Microsoft Graph page. The script follows the report's next-page link until all pages have been retrieved. It retrieves the tenant's registration report and then matches those records to the supplied group members.

The script does not create missing folders. Create the input and output directories first, and ensure the current account can read the input file and write the output file.

## Usage

From PowerShell, first authenticate to Microsoft Graph, then run:

```powershell
Connect-MgGraph
.\AuthMethods-SecurityQuestionsRegistration-GroupMembers.ps1
```

Supply the required Graph permissions when connecting. Complete any interactive sign-in flow and consent prompts before running the script.

## User Matching

For each group member, the script looks up the `Id` in the registration report's `id` field. It does not search by name, UPN, or email.

Users without a matching registration record are included in the output. For those rows, `RegistrationRecordFound` is `False` and `MethodsRegistered` is blank. The `SecurityQuestionsRegistered` value is also `False`; check `RegistrationRecordFound` to distinguish a user with no matching report record from a user whose registration record does not list Security Questions.

## Output Files

### Detailed CSV report

The configured `$OutputCsv` receives a UTF-8 CSV with these columns:

| Column | Description |
| --- | --- |
| `ObjectId` | Entra ID object ID from the input CSV |
| `DisplayName` | Display name from the input CSV |
| `UserPrincipalName` | UPN from the input CSV |
| `Mail` | Email address from the input CSV |
| `SecurityQuestionsRegistered` | `True` if the matching registration record lists `securityQuestion`; otherwise `False` |
| `MethodsRegistered` | Semicolon-separated registered authentication methods, or blank when no registration record was found |
| `RegistrationRecordFound` | `True` if a registration record was matched by object ID |

### Console summary

The script prints counts for group members, retrieved registration records, matched and unmatched users, and users with or without Security Questions. The “No Security Questions” count includes only users with a matching registration record; unmatched users are counted separately.

It also prints elapsed times for CSV import, Graph report retrieval, lookup-table creation, group comparison, CSV export, and the total run.

## Notes and Limitations

- Microsoft Graph report data may be subject to service-side availability or reporting delay.
- The script retrieves all pages of the tenant's authentication method registration report before comparing the records with the input group. Runtime depends on report size and Graph response times.
- A user with no matching registration record is not counted in either the “Security Questions” or “No Security Questions” totals.
- The script uses the current Microsoft Graph PowerShell session and does not disconnect from Graph automatically. Run `Disconnect-MgGraph` afterward if needed.
