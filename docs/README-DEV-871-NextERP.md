# DEV-871 — NextERP Account Provisioning

## Overview

DEV-871 implements automated employee account provisioning for **NextERP (ERPNext/Frappe)** through the shared n8n provisioning workflow.

The completed flow is:

```text
Employee provisioning request
        ↓
n8n Webhook
        ↓
Shared-secret validation
        ↓
NextERP REST API
        ↓
Create User
        ↓
Assign Employee role
        ↓
Shape Response
        ↓
HTTP 201 response
```

The environment is local/self-hosted as required by the DEV-868 project.

---

## 1. Jira Requirement

**DEV-871 — NextERP Account Provisioning**

The requirement was to allow an employee provisioning request to create a NextERP account with the standard **Employee** role.

The implementation needed to:

* Create a NextERP user from the employee name, email and password received by n8n.
* Assign the standard `Employee` role.
* Avoid elevated administrator/manager roles.
* Authenticate against the NextERP API.
* Return the actual login identifier.
* Reject invalid provisioning secrets with HTTP `401`.
* Document the endpoint, authentication and request structure.

---

# 2. Local NextERP Environment

NextERP was implemented using the official Frappe Docker environment.

Local directory:

```text
~/nexterp-local
```

The environment was started with:

```bash
cd ~/nexterp-local
docker compose -f pwd.yml up -d
```

NextERP is available locally at:

```text
http://localhost:8080
```

The local site name is:

```text
frontend
```

---

# 3. Installed Versions

The installed applications were verified as:

```text
frappe   16.35.0
erpnext  16.36.1
```

The NextERP frontend was confirmed reachable on port `8080`.

---

# 4. ERPNext Initial Configuration

The ERPNext setup wizard was completed for the DEV-868 project.

Configuration:

| Setting              | Value            |
| -------------------- | ---------------- |
| Company              | DEV-868          |
| Company Abbreviation | DEV              |
| Chart of Accounts    | Standard         |
| Financial Year       | 01-01-2026       |
| Team Size            | Small team       |
| Work                 | Other / Services |
| Demo Data            | Disabled         |

The local Administrator account was configured during setup.

---

# 5. API Authentication

Frappe API authentication uses:

```http
Authorization: token <api_key>:<api_secret>
```

A dedicated provisioning API user was created:

```text
n8n-provisioner@dev868.local
```

User type:

```text
System User
```

The API credentials were generated locally and successfully used to authenticate against the Frappe API.

### Security

The actual API key and API secret are intentionally **not included in this README or GitHub repository**.

They must remain outside source control.

---

# 6. API Authentication Verification

Authentication was verified using:

```text
GET /api/method/frappe.auth.get_logged_user
```

Example:

```bash
curl -s http://localhost:8080/api/method/frappe.auth.get_logged_user \
  -H "Authorization: token <API_KEY>:<API_SECRET>"
```

The API returned:

```json
{
  "message": "n8n-provisioner@dev868.local"
}
```

This confirmed that the dedicated n8n provisioning user could successfully authenticate against NextERP.

---

# 7. User Creation API

The actual NextERP/Frappe REST endpoint used for user creation is:

```text
POST /api/resource/User
```

Local endpoint:

```text
http://localhost:8080/api/resource/User
```

Authentication:

```http
Authorization: token <API_KEY>:<API_SECRET>
Content-Type: application/json
```

The provisioning request supplies:

* Employee name
* Employee email
* Password

The user is created as a:

```text
System User
```

and assigned the:

```text
Employee
```

role.

---

# 8. Employee Role

The required default role for DEV-871 is:

```text
Employee
```

The completed provisioning flow assigns the employee the standard Employee role.

Elevated roles such as:

```text
System Manager
HR Manager
Accounts Manager
```

were not assigned to the provisioned employee account.

This satisfies the least-privilege requirement for the provisioning task.

---

# 9. Module Profile

The ERPNext user configuration includes a Module Profile setting.

During the local setup, the Module Profile was left at the default configuration rather than assigning a custom restricted module profile.

The available ERPNext module areas include areas such as:

* Accounts
* HR
* Payroll
* Manufacturing
* Other ERPNext modules

Module visibility is separate from the user's role assignment.

The required DEV-871 authorization control is the `Employee` role rather than administrator/manager roles.

---

# 10. n8n Workflow

The DEV-871 workflow follows the common DEV-868 provisioning architecture:

```text
Webhook
   ↓
Check Secret
   ↓
Create Account
   ↓
Shape Response
   ↓
Respond Success
```

The unauthorized branch is:

```text
Check Secret
   ↓
Respond Unauthorized
```

The workflow uses the shared provisioning-secret mechanism established in DEV-868.

---

# 11. Webhook

The shared provisioning webhook path is:

```text
/provision-account
```

The expected request body is:

```json
{
  "employeeName": "Jane Doe",
  "employeeEmail": "jane.doe@company.com",
  "password": "TempPass123!"
}
```

The request includes:

```http
x-provisioning-key: <PROVISIONING_SECRET>
```

---

# 12. Successful Provisioning

The standard DEV-868 successful request is:

```bash
curl -i -X POST http://localhost:5678/webhook/provision-account \
  -H "Content-Type: application/json" \
  -H "x-provisioning-key: $PROVISIONING_SECRET" \
  -d '{
        "employeeName": "Jane Doe",
        "employeeEmail": "jane.doe@company.com",
        "password": "TempPass123!"
      }'
```

Expected result:

```text
HTTP 201
```

with a response containing the actual login identifier:

```json
{
  "username": "..."
}
```

The DEV-871 workflow was successfully tested end-to-end.

---

# 13. Unauthorized Request

The invalid-secret test was also completed.

Example:

```bash
curl -i -X POST http://localhost:5678/webhook/provision-account \
  -H "Content-Type: application/json" \
  -H "x-provisioning-key: wrong-secret" \
  -d '{
        "employeeName": "Jane Doe",
        "employeeEmail": "jane.doe@company.com",
        "password": "TempPass123!"
      }'
```

Expected and verified result:

```text
HTTP 401
```

Response:

```json
{
  "error": "Unauthorized"
}
```

---

# 14. Security Configuration

The provisioning secret is not hardcoded into the workflow.

The n8n environment uses:

```text
PROVISIONING_SECRET
```

and:

```text
PROVISIONING_SECRET_REQUIRED=true
```

n8n also uses:

```text
N8N_BLOCK_ENV_ACCESS_IN_NODE=false
```

The n8n data directory is persisted using:

```text
n8n_data:/home/node/.n8n
```

No real API credentials, passwords or provisioning secrets should be committed to GitHub.

---

# 15. End-to-End Result

The completed DEV-871 provisioning flow is:

```text
Employee
   │
   │ employeeName
   │ employeeEmail
   │ password
   ▼
n8n Webhook
   │
   ▼
Shared Secret Validation
   │
   ├── Invalid → HTTP 401
   │
   ▼
NextERP API
POST /api/resource/User
   │
   ▼
System User Created
   │
   ▼
Employee Role Assigned
   │
   ▼
Response Shaped
   │
   ▼
HTTP 201
{
  "username": "..."
}
```

---

# 16. Verification

DEV-871 was verified at both the automation and application levels.

### n8n

Confirmed:

* Webhook received the provisioning request.
* Shared secret was validated.
* NextERP API request succeeded.
* Workflow completed successfully.
* Successful response returned the login identifier.

### NextERP

Confirmed:

* Provisioned user was created.
* User exists in the NextERP administration interface.
* Employee role was assigned.
* Elevated administrator/manager roles were not assigned.

---

# 17. Evidence

The following evidence was captured.

### Successful n8n execution

```text
DEV-871-NextERP-successful-execution.png
```

Shows the successful provisioning workflow execution and response.

### Provisioned NextERP user

```text
DEV-871-NextERP-user-provisioned.png
```

Shows the provisioned employee account inside NextERP.

---

# 18. DEV-871 Completion Status

| Requirement               | Status              |
| ------------------------- | ------------------- |
| Local NextERP environment | ✅ Complete          |
| ERPNext setup             | ✅ Complete          |
| API authentication        | ✅ Complete          |
| User creation API         | ✅ Verified          |
| Employee role             | ✅ Verified          |
| Least-privilege account   | ✅ Verified          |
| n8n workflow              | ✅ Complete          |
| Successful provisioning   | ✅ Verified          |
| Invalid secret            | ✅ HTTP 401 verified |
| NextERP UI verification   | ✅ Complete          |
| Evidence screenshots      | ✅ Captured          |
| Documentation             | ✅ Complete          |

---

# 19. Files and Environment

Main DEV-868 project repository:

```text
~/flowable-provisioning-source
```

Local NextERP environment:

```text
~/nexterp-local
```

NextERP:

```text
http://localhost:8080
```

n8n:

```text
http://localhost:5678
```

---

# 20. GitHub Security

Before the final DEV-868 GitHub push, ensure the repository does **not** contain:

* NextERP API keys
* NextERP API secrets
* Administrator passwords
* `PROVISIONING_SECRET`
* n8n credentials
* `.env` files containing real secrets
* Docker volumes
* database files
* local application data
* personal credentials

Only configuration templates and documentation should be committed.

---

# 21. Final Status

**DEV-871 — NextERP Account Provisioning: COMPLETE ✅**

The local NextERP application was successfully integrated with n8n, authenticated through the Frappe API, provisioned an employee account with the required `Employee` role, rejected unauthorized provisioning requests, and was verified in the NextERP UI.

**Next project task: DEV-872 — Superset Account Provisioning.**
