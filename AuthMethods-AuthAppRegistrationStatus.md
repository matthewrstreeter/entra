# AuthMethods-AuthAppRegistrationStatus

## Overview
This PowerShell script audits Microsoft Authenticator App registration status for members of a security group in Microsoft Entra ID. It generates a comprehensive CSV report showing which users have the Microsoft Authenticator authentication method registered.
Script: https://github.com/matthewrstreeter/entra/blob/main/AuthMethods-AuthAppRegistrationStatus.ps1

## Purpose
- Identify users in a security group who have registered Microsoft Authenticator
- Generate audit reports for compliance and security purposes
- Track user authentication method adoption across your organization

## Prerequisites

### Required PowerShell Modules
```powershell
Install-Module Microsoft.Graph.Groups
Install-Module Microsoft.Graph.Users.Authentication
```

### Required Microsoft Graph Scopes
- `Group.Read.All` – Read group memberships
- `User.Read.All` – Read basic user profiles
- `UserAuthenticationMethod.Read.All` – Read user authentication methods
- `AuditLog.Read.All` – Read sign-in activity

### Required Permissions
Your account must have sufficient permissions in Microsoft Entra ID to:
- Read security group memberships
- Read user authentication methods
- Read user sign-in activity

## Parameters

### GroupName
- **Type:** String
- **Required:** Yes
- **Description:** The display name of the security group to audit

### OutputCsvPath
- **Type:** String
- **Required:** Yes
- **Description:** Full path where the CSV report will be saved (e.g., `C:\Reports\report.csv`)

## Usage

```powershell
.\AuthMethods-AuthAppRegistrationStatus.ps1 -GroupName "Security Team" -OutputCsvPath "C:\Reports\auth_report.csv"
```

## Output

The script generates a CSV file with the following columns:

| Column | Description |
|--------|-------------|
| DisplayName | User's display name in Entra ID |
| UserPrincipalName | User's email/UPN |
| UserId | Entra ID object ID |
| ManagerName | Name of the user's manager |
| ManagerEmail | Email of the user's manager |
| AuthenticatorStatus | "Registered" or "Not Registered" |
| LastSignInDateTime | Last sign-in date/time |
| LastSuccessfulSignInDateTime | Last successful sign-in date/time |
| AccountCreated | Date the account was created |
| Group | Name of the audited security group |

## Features

- ✅ Automatic Microsoft Graph connection with required scopes
- ✅ Color-coded console output for easy monitoring
- ✅ Handles pagination for large groups
- ✅ Includes manager information
- ✅ Captures sign-in activity
- ✅ Error handling with informative messages
- ✅ UTF-8 encoded CSV export

## Error Handling

The script includes error handling for:
- Failed Microsoft Graph connections
- Security group not found
- Inaccessible user accounts
- Missing manager assignments
- Authentication method retrieval failures

## Notes

- The script focuses on `#microsoft.graph.microsoftAuthenticatorAuthenticationMethod` registration
- Users with Passwordless Phone Sign-in may appear as registered
- The script automatically disconnects from Microsoft Graph upon completion
- If a user's manager is not assigned or inaccessible, "N/A" is recorded

## Version History

- **V1.0** (16-Apr-2026) – Initial version
- **V1.1** (16-Apr-2026) – Moved GroupName and OutputCsvPath to parameters
