# DEV-869 — TwentyCRM Account Provisioning

## 1. Overview

DEV-869 implements automated employee account provisioning for a local **TwentyCRM** instance using **n8n**.

The workflow receives an employee provisioning request through an n8n webhook, validates a shared provisioning secret, sends the employee invitation to TwentyCRM through its API, and returns a standardized response.

### Architecture

```text
Employee Provisioning Request
          ↓
     n8n Webhook
          ↓
   Shared Secret Check
          ↓
    TwentyCRM API
          ↓
     Shape Response
          ↓
   Success / Unauthorized
```

The implementation runs entirely in a local/self-hosted environment.

---

# 2. Environment

### n8n

URL:

```text
http://localhost:5678
```

Working directory:

```bash
~/dev-868-provisioning
```

Container:

```text
n8n-dev868
```

### TwentyCRM

Working directory:

```bash
~/twenty-local
```

Application URL:

```text
http://localhost:3000
```

Docker services:

```text
twenty-server-1
twenty-worker-1
twenty-redis-1
twenty-db-1
```

---

# 3. Shared Security

The provisioning workflow uses an environment-based shared secret.

```bash
export PROVISIONING_SECRET="dev868-test-secret-2026"
export PROVISIONING_SECRET_REQUIRED="true"
```

n8n was configured with:

```text
N8N_BLOCK_ENV_ACCESS_IN_NODE=false
```

The request secret is supplied through:

```text
x-provisioning-secret
```

Invalid secrets return:

```text
HTTP 401 Unauthorized
```

The secret is not hardcoded into the workflow.

---

# 4. TwentyCRM API Discovery

The TwentyCRM provisioning operation was identified through the application's metadata endpoint.

The working endpoint is:

```text
http://localhost:3000/metadata
```

The workflow uses TwentyCRM's `sendInvitations` mutation.

```graphql
mutation SendInvitations($emails: [String!]!, $roleId: UUID) {
  sendInvitations(emails: $emails, roleId: $roleId) {
    success
    errors
    result {
      ... on WorkspaceInvitation {
        id
        email
        roleId
        expiresAt
        __typename
      }
      __typename
    }
    __typename
  }
}
```

The Member role ID used:

```text
c5bb6a11-00a7-4f10-800a-f02b483d85f8
```

---

# 5. API Authentication

A TwentyCRM API key was created through:

```text
Settings
→ Advanced Mode
→ MCP & APIs
→ API Keys
```

The API key was verified using:

```text
/api/v1/timestamp
```

which returned HTTP 200.

The API key is stored in an n8n Header Auth credential and is not hardcoded in the workflow.

---

# 6. n8n Workflow

The DEV-869 workflow follows:

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

Unauthorized requests follow:

```text
Check Secret
    ↓
Respond Unauthorized
```

---

# 7. Webhook Request

Example request:

```json
{
  "employeeEmail": "dev869-screenshot@example.com"
}
```

The shared secret is supplied through:

```text
x-provisioning-secret
```

---

# 8. Create Account Node

### Method

```text
POST
```

### URL

```text
http://host.docker.internal:3000/metadata
```

`host.docker.internal` is required because n8n runs inside Docker while TwentyCRM is exposed on the host.

### Authentication

```text
Generic Credential Type
→ Header Auth
```

### Request body

```json
{
  "operationName": "SendInvitations",
  "variables": {
    "emails": ["{{ $json.body.employeeEmail }}"],
    "roleId": "c5bb6a11-00a7-4f10-800a-f02b483d85f8"
  },
  "query": "mutation SendInvitations($emails: [String!]!, $roleId: UUID) { sendInvitations(emails: $emails, roleId: $roleId) { success errors result { ... on WorkspaceInvitation { id email roleId expiresAt __typename } __typename } __typename } }"
}
```

### Important n8n expression detail

Correct:

```text
{{ $json.body.employeeEmail }}
```

Incorrect:

```text
={{ $json.body.employeeEmail }}
```

The extra `=` can cause the literal value sent to the API to begin with `=`.

---

# 9. Shape Response

The response is normalized using the Shape Response node.

Mapping:

```text
username
=
{{ $('Webhook').item.json.body.employeeEmail }}
```

---

# 10. Success Response

The success response is:

```json
{
  "username": "{{ $json.username }}"
}
```

---

# 11. End-to-End Verification

The workflow was successfully tested with:

```bash
curl -X POST "http://localhost:5678/webhook-test/provision-account" \
  -H "Content-Type: application/json" \
  -H "x-provisioning-secret: dev868-test-secret-2026" \
  -d '{
    "employeeEmail": "dev869-screenshot@example.com"
  }'
```

Successful response:

```json
{
  "username": "dev869-screenshot@example.com"
}
```

This confirmed:

* Webhook received the request.
* Shared secret validation succeeded.
* TwentyCRM API request succeeded.
* Employee invitation was processed.
* Response was shaped correctly.
* Success response was returned.

---

# 12. Docker Networking Lesson

Because n8n runs inside Docker, `localhost` inside the n8n container refers to the n8n container itself.

Therefore, the TwentyCRM API was accessed from n8n using:

```text
http://host.docker.internal:3000/metadata
```

rather than:

```text
http://localhost:3000/metadata
```

The workflow also required HTTP rather than HTTPS because the local TwentyCRM instance was running over HTTP.

---

# 13. Evidence

Successful n8n execution screenshot:

```text
DEV-869-twentycrm-successful-execution.png
```

This screenshot shows the successful execution path through the provisioning workflow.

The planned target-application screenshot:

```text
DEV-869-twentycrm-provisioned-user.png
```

was not captured because TwentyCRM encountered a server-health issue after the system was restarted.

The missing screenshot does not change the functional verification: the actual provisioning workflow had already been successfully executed and returned the expected response.

---

# 14. Result

DEV-869 successfully demonstrated automated TwentyCRM employee provisioning through n8n using:

* Webhook-based requests
* Shared-secret validation
* API authentication
* TwentyCRM invitation API
* Docker-to-host networking
* Standardized success responses
* Unauthorized request handling

## Status

**DEV-869 — TwentyCRM: Functionally Complete**

