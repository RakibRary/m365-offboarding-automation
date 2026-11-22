# M365 Offboarding Automation

Offboards a Microsoft 365 user end to end when an agent ticks one checkbox on a Zendesk ticket.
A Zendesk trigger calls a Power Automate flow, which runs an Azure Automation runbook. The runbook locks the account, converts the mailbox to shared, removes licenses and groups, and checks its own work. The flow then posts a full summary back to the ticket as an internal note.

![M365 offboarding flow: Zendesk ticket, trigger and webhook, Power Automate, managed identity permissions, runbook lock, shared mailbox, strip and verify, note on ticket](docs/images/m365-offboarding-process.png)

## The problem

Offboarding a user by hand meant visiting several admin centers for every person: block sign-in, convert the mailbox, remove licenses, remove groups, rename the account, and then write it all up in the ticket.
It was slow, easy to get wrong, and the steps weren't recorded the same way every time.

## The result

- **One click:** an agent ticks **Offboard to M365** on the ticket, and the rest happens automatically.
- **Same order every time,** including the timing-sensitive mailbox conversion.
- **A built-in audit trail:** every run writes a summary to the ticket listing what changed, what was skipped and what needs manual attention.
- **No stored passwords or app secrets:** the runbook signs in with an Azure managed identity.

## What happens to each user

| Step | Action |
|---|---|
| 1 | Block sign-in (`accountEnabled = false`) |
| 2 | Revoke all sign-in sessions |
| 3 | Rename the display name to `X-<Display Name>` |
| 4 | Convert the mailbox to a **shared mailbox** and hide it from the GAL, retrying until the change sticks |
| 5 | Remove **directly assigned** licenses (group-based licenses drop when the user leaves the licensing group) |
| 6 | Remove the user from cloud groups. Dynamic, on-prem-synced and mail-enabled groups are listed for manual follow-up |
| 7 | **Verification pass:** check again that the mailbox is shared and hidden, and fix it if not |
| 8 | Post the summary to the Zendesk ticket as an internal note |

Each step checks the current state first, so running it twice on the same user is safe.

## Architecture

```
Zendesk ticket (offboarding form + approval checkbox)
   │  trigger: Form = Offboarding AND checkbox is checked
   ▼
Zendesk webhook ── POST { "upn", "ticket_id" } + X-Offboard-Secret header
   ▼
Power Automate flow
   ├─ check the secret header
   ├─ Azure Automation: Create job (wait for completion)
   ├─ Azure Automation: Get job output
   └─ HTTP PUT → Zendesk ticket internal note
   ▼
Azure Automation runbook (PowerShell 5.1, system-assigned managed identity)
   ├─ Microsoft Graph   → sign-in, sessions, rename, licenses, groups
   └─ Exchange Online   → shared mailbox, GAL hide, verification
```

## Repo layout

```
runbook/       Azure Automation runbook (PowerShell 5.1)
scripts/       Standalone bulk version: run it locally against a list of UPNs, with -WhatIf
docs/images/   Diagram
```

## Setup

### 1. Azure Automation account
1. Create an Automation account and leave the **system-assigned managed identity** turned on.
2. Copy the identity's **Object (principal) ID**.

### 2. Modules (all Runtime **5.1**)
Import these from the gallery, in this order, waiting for each to show **Available**:
`PackageManagement` → `PowerShellGet` → `ExchangeOnlineManagement`. Check that `Az.Accounts` is present.

> Don't use Runtime 7.2. ExchangeOnlineManagement fails there with an assembly error.

### 3. Give the managed identity its permissions
Run this from a compliant Windows machine. Conditional Access often blocks Cloud Shell and device-code sign-in.

```powershell
Connect-MgGraph -Scopes "AppRoleAssignment.ReadWrite.All","Application.Read.All"
$miId  = "<MANAGED_IDENTITY_OBJECT_ID>"
$graph = Get-MgServicePrincipal -Filter "AppId eq '00000003-0000-0000-c000-000000000000'"
foreach ($perm in "User.ReadWrite.All","Group.ReadWrite.All") {
  $role = $graph.AppRoles | ? Value -eq $perm
  New-MgServicePrincipalAppRoleAssignment -ServicePrincipalId $miId -PrincipalId $miId -ResourceId $graph.Id -AppRoleId $role.Id
}
$exo  = Get-MgServicePrincipal -Filter "AppId eq '00000002-0000-0ff1-ce00-000000000000'"
$role = $exo.AppRoles | ? Value -eq "Exchange.ManageAsApp"
New-MgServicePrincipalAppRoleAssignment -ServicePrincipalId $miId -PrincipalId $miId -ResourceId $exo.Id -AppRoleId $role.Id
```

Then assign the identity the **Exchange Administrator** role in Entra (Roles & admins). New permissions can take 15–60 minutes to take effect.

### 4. Runbook
Create a **PowerShell 5.1** runbook and paste in the script from `runbook/`. Set your Exchange org domain on the `Connect-ExchangeOnline -ManagedIdentity -Organization "<yourtenant>.onmicrosoft.com"` line, then **Publish**.
Test it from the portal against an account that's already been offboarded.

### 5. Power Automate flow (needs a Premium license)
1. Trigger: **When an HTTP request is received**, with the schema `{ "upn": "", "ticket_id": "" }`.
2. Initialize the variables `Upn` = `triggerBody()?['upn']` and `TicketID` = `triggerBody()?['ticket_id']`. Put these above the condition.
3. **Condition:** `triggerOutputs()?['headers']?['x-offboard-secret']` equals `<YOUR_SECRET>`. Use the lowercase header name.
4. Under **True:** Azure Automation **Create job** (wait for the job = Yes, with `Upn` as the parameter), then **Get job output**.
5. **HTTP PUT** to `https://<subdomain>.zendesk.com/api/v2/tickets/@{variables('TicketID')}.json`, using Basic auth (`agent@example.com/token` plus your API token) and this body:
   ```json
   { "ticket": { "comment": { "html_body": "@{body('Get_job_output')}", "public": false } } }
   ```
6. Save, then copy the trigger's **HTTP POST URL**.

### 6. Zendesk webhook
Go to Admin Center → Webhooks → **Create webhook** (Trigger or automation). Use the flow URL, POST and JSON, and add the header `X-Offboard-Secret: <YOUR_SECRET>`.

### 7. Zendesk checkbox and trigger
1. Add a checkbox field (**Offboard to M365**) that agents can edit, and put it on the offboarding form.
2. Create a trigger with the conditions **Form is Offboarding** and **Offboard to M365 is checked**. Set its action to **Notify webhook**:
   ```json
   { "upn": "{{ticket.ticket_field_<UPN_FIELD_ID>}}", "ticket_id": "{{ticket.id}}" }
   ```

## Common problems

| Symptom | Fix |
|---|---|
| Exchange module assembly error | The runbook and modules must use Runtime **5.1** |
| Secret check always false | Use the **lowercase** header name in the condition |
| Variable contains the literal text `triggerBody()` | Enter it as an expression, not plain text |
| Mailbox not shared after the run | The runbook converts **before** removing licenses, with retries and a final verification. Check the MAILBOX STATUS line in the ticket note |
| 403 on Graph or Exchange calls | Wait for permissions to take effect, and check the Exchange Administrator role |
| Can't find the shared mailbox in the admin center | It's hidden from the GAL. Search for the exact `X-` name, or use `Get-Mailbox -RecipientTypeDetails SharedMailbox` |

## Notes and limits

- **Hybrid accounts:** for users synced from on-prem AD, block sign-in and change groups in AD as well. Otherwise the next sync can overwrite the cloud changes.
- **Large or held mailboxes:** shared mailboxes over 50 GB, or with an archive or litigation hold, still need a license. Leave those users out of license removal.
- **Human approval:** the trigger only fires when an agent ticks the checkbox. Never run it straight from a form submission.

## Security

- The runbook uses a **managed identity**, so no client secret is stored anywhere.
- Store the webhook secret and Zendesk API token only in Power Automate and Zendesk. Never commit them.
- All values in this repo (`<YOUR_SECRET>`, `<subdomain>`, `<yourtenant>`, field IDs) are placeholders.
- Test only on accounts that have already been offboarded.
