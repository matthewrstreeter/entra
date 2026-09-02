# Entra ID Authentication Method Reports

This directory contains PowerShell scripts for reporting authentication method registration status in Microsoft Entra ID.

## AuthMethods-SecurityQuestionsRegistration.ps1

`AuthMethods-SecurityQuestionsRegistration.ps1` checks whether users in an input CSV have registered **Security Questions** as an authentication method. It looks up each user in Entra ID, queries the Microsoft Graph authentication method registration details report, and writes both a detailed CSV report and a text summary.

## Prerequisites

- PowerShell 7 or Windows PowerShell
- Microsoft Graph PowerShell SDK with the following cmdlets available:
  - `Connect-MgGraph`
  - `Get-MgUser`
  - `Get-MgReportAuthenticationMethodUserRegistrationDetail`
- An account with Microsoft Graph permissions to read users and authentication method registration reports
- An input CSV containing the required columns described below
- Existing output directories for the CSV and summary files

The script connects with these scopes:

- `User.Read.All`
- `AuditLog.Read.All`

Depending on tenant policies, an administrator may need to grant consent for these delegated permissions.

## Input CSV

The input CSV must contain these headers:

| Header | Required | Description |
| --- | --- | --- |
| `Employee ID` | Recommended | Used first to match `onPremisesSamAccountName` in Entra ID |
| `First Name` | Required for fallback | Used with `Last Name` when the Employee ID lookup does not find a user |
| `Last Name` | Required for fallback | Used with `First Name` when the Employee ID lookup does not find a user |

Example:

```csv
Employee ID,First Name,Last Name
12345,Alex,Example
67890,Jordan,O'Brien
```

The script also accounts for a possible UTF-8 BOM or hidden characters in the `Employee ID` header. Names containing apostrophes are escaped before being used in the Microsoft Graph filter.

## Configuration

Open the script and update these variables before running it:

```powershell
$csvInputPath = "\\Path\\To\\Input\\UsersList.csv"
$csvOutputPath = "\\Path\\To\\Output\\UsersList-SecurityQuestionsReport.csv"
$summaryOutputPath = "\\Path\\To\\Output\\UsersList-SecurityQuestionsSummary.txt"
```

The script does not create missing folders. Create the input and output directories first, and ensure the current account can read the input file and write the output files.

## Usage

From PowerShell, run:

```powershell
.\\AuthMethods-SecurityQuestionsRegistration.ps1
```

On first use, `Connect-MgGraph` opens an interactive sign-in flow. Complete authentication with an account that has the required permissions.

## User Matching

Each CSV row is processed in this order:

1. Match `Employee ID` against the Entra ID `onPremisesSamAccountName` property.
2. If no match is found, match `First Name` and `Last Name` against `givenName` and `surname`.
3. If neither lookup succeeds, record the user as `Not Found`.

When more than one user matches a filter, the script uses the first returned user. Name matching should therefore be treated as a fallback and reviewed for possible duplicates.

Rows with both a blank Employee ID and a blank First Name are skipped.

## Output Files

### Detailed CSV report

The configured `$csvOutputPath` receives a UTF-8 CSV with these columns:

| Column | Description |
| --- | --- |
| `Employee ID` | Employee ID from the input row |
| `MatchMethod` | `EE ID`, `Name Match`, or `Not Found` |
| `DisplayName` | Display name returned by Entra ID, or the input name for an unmatched row |
| `UserPrincipalName` | Entra ID UPN, or `Not Found in Entra` |
| `HasSecurityQuestions` | `True`, `False`, or `N/A` for an unmatched user |

`HasSecurityQuestions` is `True` when the registration report's `MethodsRegistered` collection contains `securityQuestion`. A missing registration detail or a collection without that value is reported as `False`.

### Text summary

The configured `$summaryOutputPath` receives a text summary containing:

- Total users in the imported CSV
- Number with security questions registered
- Number without security questions registered
- Number not found in Entra ID, when applicable
- Number matched by first and last name, when applicable

The script also prints progress and the same summary to the console using color-coded status messages.

## Notes and Limitations

- Microsoft Graph report data may be subject to service-side availability or reporting delay.
- The script suppresses errors from individual Graph lookup and report queries with `-ErrorAction SilentlyContinue`; review the output for missing or unexpected results.
- The percentage in the summary uses the total number of imported CSV rows as its denominator. Rows skipped because both Employee ID and First Name are blank are not included in the report but are still included in that total.
- The script uses delegated Graph authentication and does not disconnect from Microsoft Graph automatically. Run `Disconnect-MgGraph` afterward if needed.

## Version History

- **V1.0** (26-Aug-2026) - Initial version
