# Entra

This folder contains PowerShell scripts related to Microsoft Entra ID (formerly Azure AD) identity management tasks.

## Table of Contents

- [CA-ExportPolices_ResolveGuids.ps1](#ca-exportpolices_resolveguidps1)
- [EnterpriseApps-GetSamlConfigs.ps1](#enterpriseapps-getsamlconfigsps1)
- [AuthMethods-AuthAppRegistrationStatus.ps1](#authmethods-authappregistrationstatusps1)
- [AuthMethods-SecurityQuestionsRegistration-UserList.ps1](#authmethods-securityquestionsregistration-userlistps1)
- [AuthMethods-SecurityQuestionsRegistration-GroupMembers.ps1](#authmethods-securityquestionsregistration-groupmembersps1)
- [AuthMethods-SecurityQuestionsRegistration-GroupAuto.ps1](#authmethods-securityquestionsregistration-groupautops1)
- [PIM-EligibleRolesActivateRoles.ps1](#pim-eligiblerolesactivaterolesps1)

---

<h2><u>Scripts</u></h2>

### CA-ExportPolices_ResolveGuids.ps1

Exports Microsoft Entra Conditional Access policies as individual JSON files and resolves referenced object IDs into human-readable names where possible. Original JSON exports are preserved, and the script also creates resolved JSON copies, a detailed assignments CSV, and a policy summary CSV.

**Prerequisites:**
- Microsoft.Graph.Authentication module
- Microsoft.Graph.Identity.SignIns module
- Delegated Microsoft Graph permissions: Policy.Read.All, Directory.Read.All, Application.Read.All

**Parameters:**
- `OutputFolder`: The output directory (default: `./Temp/CAPolicies`)

**Usage:**
```powershell
.\Entra\CA-ExportPolices_ResolveGuids.ps1
```

Use `-OutputFolder` to choose a different output directory. The output includes `Original/` and `Resolved/` JSON folders, `ConditionalAccessPolicy-ResolvedAssignments.csv`, and `ConditionalAccessPolicy-PolicySummary.csv`.

---

### EnterpriseApps-GetSamlConfigs.ps1

Retrieves SAML configuration details for all enterprise applications configured for SAML single sign-on (SSO) in Microsoft Entra ID. The script collects information such as display name, app ID, login/logout URLs, reply URLs, and token signing key validity, then exports the data to a CSV file.

**Prerequisites:**
- Microsoft.Graph.Applications module
- Microsoft.Graph.Beta.Applications module
- Microsoft.Graph.Identity.SignIns module
- Appropriate permissions: Application.Read.All

**Usage:**
- Run the script to generate a CSV file in the `.\Temp\` directory with a timestamped filename.

---

### AuthMethods-AuthAppRegistrationStatus.ps1

Checks the registration status of the Microsoft Authenticator app as an authentication method for members of a specified security group. The script retrieves user details, manager information, and authentication method status, then exports the results to a CSV file.

**Prerequisites:**
- Microsoft.Graph.Groups module
- Microsoft.Graph.Users.Authentication module
- Appropriate permissions: Group.Read.All, User.Read.All, UserAuthenticationMethod.Read.All, AuditLog.Read.All

**Parameters:**
- `GroupName`: The display name of the security group to check.
- `OutputCsvPath`: The path where the CSV output should be saved.

**Usage:**
- Run the script with the required parameters to generate a report on authenticator registration status.

---

### AuthMethods-SecurityQuestionsRegistration-UserList.ps1

Checks a custom CSV user list for Security Questions registration. It matches users by Employee ID first, falls back to first and last name, and exports a detailed CSV report plus a text summary.

**Prerequisites:**
- PowerShell 7 or Windows PowerShell
- Microsoft Graph PowerShell SDK with `Connect-MgGraph`, `Get-MgUser`, and `Get-MgReportAuthenticationMethodUserRegistrationDetail` available
- Microsoft Graph permissions: `User.Read.All` and `AuditLog.Read.All`
- An input CSV with `Employee ID`, `First Name`, and `Last Name` columns
- Existing input and output directories configured in the script

Update the input and output paths in the script before running it:

```powershell
.\AuthMethods-SecurityQuestionsRegistration-UserList.ps1
```

See [AuthMethods-SecurityQuestionsRegistration-UserList.md](./AuthMethods-SecurityQuestionsRegistration-UserList.md) for CSV format, configuration, and report details.

---

### AuthMethods-SecurityQuestionsRegistration-GroupMembers.ps1

Checks members from an exported Entra group-membership CSV against the Microsoft Graph authentication method registration report, matching users by Entra object ID. It exports a detailed CSV with registration status and whether a registration record was found.

**Prerequisites:**
- PowerShell 7 or Windows PowerShell
- Microsoft Graph PowerShell SDK with `Invoke-MgGraphRequest` available
- An authenticated Microsoft Graph session with permission to read authentication method registration details
- A group-membership CSV with `Id`, `userPrincipalName`, `displayName`, and `mail` columns

Update the input and output paths in the script, authenticate to Graph, and run it:

```powershell
Connect-MgGraph
.\AuthMethods-SecurityQuestionsRegistration-GroupMembers.ps1
```

See [AuthMethods-SecurityQuestionsRegistration-GroupMembers.md](./AuthMethods-SecurityQuestionsRegistration-GroupMembers.md) for the input format, output columns, and matching details.

---

### AuthMethods-SecurityQuestionsRegistration-GroupAuto.ps1

Resolves an Entra group, retrieves its direct user members, and checks their Security Questions registration status. It can export the membership list, a registration report, and a store-level follow-up report. Optional LDAPS enrichment adds on-premises Active Directory password-expiration information.

**Prerequisites:**
- PowerShell 7.0 or later
- Microsoft Graph PowerShell Authentication module
- Delegated Graph permissions: `Group.Read.All`, `User.Read.All`, and `AuditLog.Read.All`
- An Entra role that permits reading authentication method registration reports, such as Reports Reader

The script connects to Graph if no session exists and checks required scopes on an existing delegated session. The `GROUPNAME` and `\Path\To\EntraReports` values in the script are sanitized placeholders; replace them or pass `-GroupName` and `-OutputDirectory`. Run, for example:

```powershell
.\AuthMethods-SecurityQuestionsRegistration-GroupAuto.ps1 `
    -GroupName "Example Group" `
    -OutputDirectory "C:\EntraReports"
```

See [AuthMethods-SecurityQuestionsRegistration-GroupAuto.md](./AuthMethods-SecurityQuestionsRegistration-GroupAuto.md) for parameters, output details, account-status filtering, and optional LDAPS configuration.

---

### PIM-EligibleRolesActivateRoles.ps1

Automates the activation of eligible Privileged Identity Management (PIM) roles for the current user in Microsoft Entra ID. The script connects to Microsoft Graph, retrieves the user's eligible roles, allows interactive selection of roles to activate, and submits activation requests with specified justification and duration.

**Prerequisites:**
- PowerShell 5.1 or later
- Microsoft.Graph PowerShell module
- Appropriate permissions: User.Read.All, RoleManagement.ReadWrite.Directory
- Must have eligible PIM roles assigned

**Usage:**
1. Connect to Microsoft Graph with the required scopes:
   ```powershell
   Connect-MgGraph -Scopes "User.Read.All", "RoleManagement.ReadWrite.Directory"
   ```
2. Run the script:
   ```powershell
   .\PIM-EligibleRolesActivateRoles.ps1
   ```
3. Follow the interactive prompts to select roles, provide justification, and specify activation duration.

**Interactive Parameters:**
- **Role Selection**: Choose from listed eligible roles by entering index numbers (comma-separated) or 'A' for all.
- **Justification**: A required reason for activating the roles.
- **Duration**: Number of hours for activation (default is 8 hours).
