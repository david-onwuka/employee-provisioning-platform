# DEV-872 — Superset Account Provisioning

## Project

**DEV-868 — Automate Account Provisioning via n8n**

## Subtask

**DEV-872 — Superset Account Provisioning**

## Status

**COMPLETED**

---

# 1. Objective

Implement automated Superset account provisioning using the existing local n8n provisioning workflow.

The workflow should:

1. Receive an employee provisioning request through an n8n Webhook.
2. Validate the shared provisioning secret.
3. Authenticate to Apache Superset using the Security API.
4. Obtain a JWT access token.
5. Create the employee's Superset account.
6. Assign the **Gamma** role by default.
7. Return the created username.
8. Reject requests with an invalid provisioning secret with HTTP `401 Unauthorized`.

---

# 2. Architecture

The completed provisioning flow is:

```text
Employee Provisioning Request
          │
          ▼
     n8n Webhook
          │
          ▼
     Check Secret
       /       \
    Valid     Invalid
      │          │
      ▼          ▼
Superset Login   401 Unauthorized
      │
      ▼
Create Superset Account
      │
      ▼
  Shape Response
      │
      ▼
 Respond Success
      │
      ▼
 HTTP 201 Created
```

---

# 3. Local Environment

## n8n

```text
URL: http://localhost:5678
```

Docker image:

```text
n8nio/n8n
```

Persistent n8n storage:

```text
n8n_data:/home/node/.n8n
```

n8n was run with:

```bash
docker run -it --rm -p 5678:5678 \
  -v n8n_data:/home/node/.n8n \
  -e N8N_BLOCK_ENV_ACCESS_IN_NODE=false \
  -e PROVISIONING_SECRET="$S" \
  -e PROVISIONING_SECRET_REQUIRED=true \
  -e SUPERSET_ADMIN_USERNAME="$U" \
  -e SUPERSET_ADMIN_PASSWORD="$P" \
  n8nio/n8n
```

Credentials were supplied through environment variables rather than hardcoded into the workflow.

---

## Superset

Superset was deployed locally using:

```text
apache/superset:latest
```

Superset URL:

```text
http://localhost:8088
```

Superset container:

```text
superset
```

---

# 4. Superset Security API

The Superset Security API was enabled through:

```text
~/superset_config.py
```

The configuration included:

```python
FAB_ADD_SECURITY_API = True
```

The configuration was mounted into the Superset container as:

```text
/app/pythonpath/superset_config.py
```

Superset database migrations were completed using:

```bash
docker exec superset superset db upgrade
```

Superset initialization was then completed using:

```bash
docker exec superset superset init
```

---

# 5. Superset Administrator

A Superset administrator account was created for API authentication.

Username:

```text
Davidonwuka21
```

The administrator password was kept private and supplied to n8n through:

```text
SUPERSET_ADMIN_PASSWORD
```

The password was not hardcoded into the workflow.

---

# 6. Superset Roles

After Superset initialization, the available roles were verified:

```text
1: Admin
2: Public
3: Alpha
4: Gamma
5: sql_lab
```

The DEV-872 acceptance criteria require the newly provisioned account to have the **Gamma** role.

Therefore:

```text
Gamma role ID = 4
```

The n8n user creation request was configured with:

```json
"roles": [4]
```

This means the newly created account receives Gamma rather than Alpha or an administrative role.

---

# 7. Superset API Endpoints

## Login API

The workflow authenticates using:

```text
POST /api/v1/security/login
```

Final URL used by n8n:

```text
http://host.docker.internal:8088/api/v1/security/login
```

Request body:

```json
{
  "username": "<SUPERSET_ADMIN_USERNAME>",
  "password": "<SUPERSET_ADMIN_PASSWORD>",
  "provider": "db",
  "refresh": false
}
```

Superset returns a JWT access token.

---

## Create User API

The workflow then calls:

```text
POST /api/v1/security/users/
```

Final URL:

```text
http://host.docker.internal:8088/api/v1/security/users/
```

The JWT returned from the login request is supplied as:

```text
Authorization: Bearer <JWT>
```

User creation body:

```json
{
  "username": "<employeeEmail>",
  "first_name": "<employeeName>",
  "last_name": "",
  "email": "<employeeEmail>",
  "password": "<employeePassword>",
  "active": true,
  "roles": [4]
}
```

---

# 8. Docker Networking Fix

The first version of the n8n workflow used:

```text
http://localhost:8088
```

for the Superset API.

This did not work because n8n was running inside its own Docker container. From inside that container, `localhost` referred to n8n rather than the Superset container/host service.

Both Superset API URLs were therefore changed to:

```text
http://host.docker.internal:8088
```

Final endpoints:

```text
http://host.docker.internal:8088/api/v1/security/login
```

```text
http://host.docker.internal:8088/api/v1/security/users/
```

After this change, the provisioning workflow completed successfully.

---

# 9. n8n Workflow

Workflow name:

```text
Superset Account Provisioning
```

Workflow file:

```text
flowable-provisioning-superset.json
```

Project location:

```text
~/dev-868-provisioning
```

The workflow was created from the existing provisioning template and adapted specifically for Superset.

---

# 10. Workflow Nodes

The workflow contains the following main logic:

```text
Webhook
   │
   ▼
Check Secret
   │
   ├────────────── False ──────► Respond Unauthorized
   │
   ▼ True
Superset Login
   │
   ▼
Create Superset Account
   │
   ▼
Shape Response
   │
   ▼
Respond Success
```

---

# 11. Webhook

HTTP method:

```text
POST
```

Path:

```text
provision-account
```

Test URL:

```text
http://localhost:5678/webhook-test/provision-account
```

Expected header:

```text
x-provisioning-key
```

Expected request body:

```json
{
  "employeeEmail": "dev872-test@example.com",
  "employeeName": "DEV872 Test",
  "password": "Dev872Test!123"
}
```

---

# 12. Secret Validation

The workflow uses:

```text
PROVISIONING_SECRET
```

and:

```text
PROVISIONING_SECRET_REQUIRED
```

The request header:

```text
x-provisioning-key
```

is compared with:

```text
PROVISIONING_SECRET
```

If the values match, execution continues to Superset.

If they do not match, the request is sent to the unauthorized response.

---

# 13. Unauthorized Test

An invalid provisioning key was tested.

The Check Secret node correctly routed the request through the:

```text
False Branch
```

to:

```text
Respond Unauthorized
```

The resulting response was:

```text
HTTP/1.1 401 Unauthorized
```

with:

```json
{
  "error": "Unauthorized"
}
```

This confirmed that the provisioning endpoint does not allow requests with an invalid secret.

---

# 14. Superset Login

The Superset Login node performs:

```text
POST /api/v1/security/login
```

using the administrator credentials stored in:

```text
SUPERSET_ADMIN_USERNAME
SUPERSET_ADMIN_PASSWORD
```

The returned JWT is passed to the Create Superset Account node.

The Superset API login was also manually verified during setup and returned a valid JWT.

---

# 15. Create Superset Account

The Create Superset Account node calls:

```text
POST /api/v1/security/users/
```

The username and email are taken from:

```text
employeeEmail
```

The first name is taken from:

```text
employeeName
```

The requested employee password is passed to Superset.

The account is activated using:

```json
"active": true
```

The required Gamma role is assigned using:

```json
"roles": [4]
```

---

# 16. Shape Response

The Shape Response node returns the employee username.

The response value is taken from:

```text
employeeEmail
```

For the successful test:

```text
dev872-test@example.com
```

---

# 17. Successful Provisioning Test

A real end-to-end test was performed against the n8n test webhook.

Test account:

```text
Email: dev872-test@example.com
Name: DEV872 Test
```

Request:

```bash
curl -i -X POST 'http://localhost:5678/webhook-test/provision-account' \
  -H 'Content-Type: application/json' \
  -H 'x-provisioning-key: <PROVISIONING_SECRET>' \
  -d '{"employeeEmail":"dev872-test@example.com","employeeName":"DEV872 Test","password":"Dev872Test!123"}'
```

The final successful response was:

```text
HTTP/1.1 201 Created
```

Response body:

```json
{
  "username": "dev872-test@example.com"
}
```

This confirmed the complete provisioning chain was operational.

---

# 18. Successful n8n Execution

The successful provisioning appeared in the n8n Executions page as:

```text
Sep 29, 18:17:50
Succeeded in 1.365s
Execution ID: #38
```

The successful execution followed the intended provisioning path rather than the unauthorized branch.

---

# 19. Superset User Verification

After the successful provisioning request, the new account was verified directly in the Superset UI.

Navigation:

```text
Superset
→ Settings
→ List Users
```

The provisioned account:

```text
dev872-test@example.com
```

was found successfully.

The account was verified with:

```text
Gamma
```

as its assigned role.

This provided direct confirmation that the account created through n8n existed in Superset with the required role.

---

# 20. Evidence

Two screenshots were captured for DEV-872.

## Screenshot 1 — Successful n8n Provisioning

Filename:

```text
DEV-872-n8n-successful-provisioning.png
```

Purpose:

Shows the successful n8n execution for the Superset provisioning workflow.

---

## Screenshot 2 — Superset Gamma User

Filename:

```text
DEV-872-Superset-Gamma-user.png
```

Purpose:

Shows the provisioned user in:

```text
Settings → List Users
```

and confirms the Gamma role.

---

# 21. Acceptance Criteria

| Requirement                              | Status     |
| ---------------------------------------- | ---------- |
| Enable `FAB_ADD_SECURITY_API`            | ✅ Complete |
| n8n authenticates with Superset          | ✅ Complete |
| Obtain Superset JWT                      | ✅ Complete |
| Create Superset user through API         | ✅ Complete |
| Assign Gamma role                        | ✅ Complete |
| Verify Gamma-only account                | ✅ Complete |
| Shape Response returns username          | ✅ Complete |
| Successful provisioning returns HTTP 201 | ✅ Complete |
| Invalid secret returns HTTP 401          | ✅ Complete |
| Actual account verified in Superset      | ✅ Complete |
| Evidence screenshots captured            | ✅ Complete |

---

# 22. Files

Main workflow:

```text
~/dev-868-provisioning/flowable-provisioning-superset.json
```

Superset configuration:

```text
~/superset_config.py
```

Evidence:

```text
DEV-872-n8n-successful-provisioning.png
DEV-872-Superset-Gamma-user.png
```

---

# 23. Security Notes

Secrets were not hardcoded into the n8n workflow.

The workflow uses environment variables:

```text
PROVISIONING_SECRET
PROVISIONING_SECRET_REQUIRED
SUPERSET_ADMIN_USERNAME
SUPERSET_ADMIN_PASSWORD
```

The provisioning secret is supplied through:

```text
x-provisioning-key
```

Invalid provisioning keys are rejected with:

```text
401 Unauthorized
```

The actual secret and Superset administrator password should not be committed to Git or included in project documentation.

---

# 24. Final Result

DEV-872 successfully implemented automated Superset account provisioning through n8n.

The completed system can:

* receive employee provisioning requests;
* validate the shared provisioning secret;
* authenticate with Superset;
* obtain a JWT;
* create a new Superset user;
* assign the Gamma role;
* return the created username;
* return HTTP `201 Created` for successful provisioning; and
* return HTTP `401 Unauthorized` for invalid provisioning requests.

## Final Status

**DEV-872 — SUPerset Account Provisioning: COMPLETE ✅**
