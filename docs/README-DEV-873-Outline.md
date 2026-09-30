# DEV-873 — Outline Account Provisioning

## 1. Overview

DEV-873 implements automated employee account provisioning for a local **Outline** knowledge-base instance using **n8n**.

The objective was to extend the DEV-868 employee provisioning platform to Outline so that a provisioning request can be received by n8n, validated using the shared provisioning secret, sent to the Outline API, and returned as a standardized response.

The completed workflow follows the common DEV-868 provisioning architecture:

```text
Employee Provisioning Request
          ↓
     n8n Webhook
          ↓
   Shared Secret Check
          ↓
     Outline API
          ↓
     Shape Response
          ↓
   Success / Unauthorized
```

DEV-873 was completed successfully.

---

# 2. Objective

The objective of DEV-873 was to integrate Outline into the local employee account provisioning platform.

The implementation needed to demonstrate that:

* An employee provisioning request can be received by n8n.
* The request can be authenticated using the shared provisioning secret.
* n8n can authenticate with the Outline API.
* n8n can send an employee invitation through Outline.
* The employee can be created as a standard Outline member.
* The workflow can return a standardized response.
* Unauthorized requests are rejected.
* The provisioning operation can be verified in the Outline application.

---

# 3. Environment

The following local environment was used for DEV-873.

| Component                | Details                              |
| ------------------------ | ------------------------------------ |
| n8n                      | `http://localhost:5678`              |
| Outline                  | `http://localhost:3001`              |
| Outline source directory | `~/outline`                          |
| Workflow file            | `flowable-provisioning-outline.json` |
| Workflow name            | `Outline Account Provisioning`       |
| Database                 | PostgreSQL                           |
| Cache                    | Redis                                |
| Storage                  | Local Docker volume                  |
| Deployment               | Docker Compose                       |

Outline was run locally as part of the DEV-868 self-hosted development environment.

---

# 4. Outline Local Deployment

Outline was configured as a local self-hosted instance.

The source directory used for the deployment was:

```text
~/outline
```

The Outline configuration was based on the provided environment template.

The local deployment included:

* Outline application
* PostgreSQL
* Redis
* Persistent storage/database volumes

The application was configured to listen on:

```text
0.0.0.0
```

and was exposed locally through:

```text
http://localhost:3001
```

---

# 5. Outline Configuration

The local Outline environment was configured using a `.env` file based on the project's environment sample.

The configuration included the required application settings and generated secrets.

Slack placeholder configuration that was not required for the local implementation was removed.

The following important local settings were used:

```text
PGSSLMODE=disable
HOST=0.0.0.0
FORCE_HTTPS=false
```

These settings allowed the application to operate correctly in the local Docker-based development environment.

---

# 6. Generated Application Secrets

Outline requires application secrets for its local deployment.

Secrets were generated locally rather than copied from an external environment.

OpenSSL was used to generate the required secret values.

Real secret values are intentionally not documented in this repository.

The `.env` file containing the actual secrets remains local and must never be committed to Git.

---

# 7. Local Outline Administrator

A local administrator account was created for managing the Outline workspace.

The development administrator used for the setup was:

```text
DEV873 Admin
```

with the local development email:

```text
dev873-admin@example.com
```

This account was used to complete the initial Outline setup and generate the API credential required by the n8n provisioning workflow.

---

# 8. Outline API Authentication

An Outline API key was created for integration with n8n.

The API key was configured with the required permission scope:

```text
write
```

This allowed the n8n workflow to perform the employee invitation operation.

The API key was stored as a credential in n8n rather than being embedded directly into the workflow logic.

The actual API key is intentionally excluded from this repository.

---

# 9. Security Considerations

The provisioning workflow uses two separate authentication mechanisms:

### Provisioning request authentication

The incoming employee request is protected by the shared DEV-868 provisioning secret.

### Outline API authentication

The outgoing request to Outline is authenticated using an Outline API key stored in n8n credentials.

This separation prevents the incoming provisioning secret from being used as the application's API credential.

No real credentials, API keys, passwords, or secrets are committed to the repository.

---

# 10. Outline Provisioning API

The Outline API endpoint used for employee invitations was:

```text
POST /api/users.invite
```

From the n8n Docker container, the local Outline service was reached through:

```text
http://host.docker.internal:3001/api/users.invite
```

This was necessary because `localhost` inside the n8n Docker container refers to the n8n container itself, not the host machine where Outline was running.

---

# 11. Docker-to-Host Networking

One important implementation detail in DEV-873 was Docker networking.

The Outline application was available from the host at:

```text
http://localhost:3001
```

However, n8n was running inside Docker.

Therefore, using:

```text
http://localhost:3001
```

from an n8n HTTP Request node would attempt to reach port 3001 inside the n8n container.

The working configuration was:

```text
http://host.docker.internal:3001
```

This allows the n8n container to communicate with the Outline service exposed on the Docker host.

This pattern is important for the other local integrations in DEV-868 as well.

---

# 12. Outline Invitation Payload

The Outline invitation endpoint accepts an `invites` array.

The provisioning request used the following structure:

```json
{
  "invites": [
    {
      "email": "newhire@example.com",
      "name": "New Hire",
      "role": "member"
    }
  ],
  "suppressEmail": true
}
```

The important fields are:

* `email` — employee email address
* `name` — employee display name
* `role` — Outline account role
* `suppressEmail` — prevents the local development environment from attempting to send an external email

The employee was provisioned with the standard:

```text
member
```

role.

No administrative role was assigned.

---

# 13. Why `suppressEmail` Was Used

The local environment was intended for development and testing rather than production email delivery.

Therefore:

```json
"suppressEmail": true
```

was used.

This allowed the invitation operation to be tested without requiring an external SMTP/email delivery configuration.

The result could still be verified through the Outline application.

---

# 14. n8n Workflow

The completed workflow was named:

```text
Outline Account Provisioning
```

The workflow follows the reusable DEV-868 provisioning pattern.

The major nodes are:

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

Check Secret
   ↓
Respond Unauthorized
```

---

# 15. Webhook Node

The workflow begins with an n8n Webhook node.

The webhook receives the employee provisioning request.

The request contains the employee information required by the workflow.

The webhook is also protected by the shared provisioning secret.

The test endpoint used during development was:

```text
/webhook-test/provision-account
```

The production webhook path follows the standard DEV-868 provisioning path:

```text
/webhook/provision-account
```

---

# 16. Shared Secret Validation

Before making the Outline API request, the workflow validates the provisioning secret.

The secret is stored in the n8n environment rather than hardcoded into the workflow.

The workflow uses the shared DEV-868 provisioning security model.

The validation logic ensures that an invalid provisioning request does not continue to the Outline API.

---

# 17. Unauthorized Branch

If the provisioning secret does not match the configured secret, the workflow follows the unauthorized branch.

The expected response is:

```text
HTTP 401 Unauthorized
```

This prevents unauthorized callers from triggering account provisioning.

The same security pattern is used throughout the DEV-868 application integrations.

---

# 18. Create Account Node

The Create Account node sends the provisioning request to Outline.

The configured endpoint is:

```text
http://host.docker.internal:3001/api/users.invite
```

The request uses Bearer authentication with the Outline API key stored in n8n credentials.

The request body contains the employee invitation details.

The final test used:

```text
dev873-test@example.com
```

with the employee name:

```text
DEV873 Test
```

and the role:

```text
member
```

The email notification was suppressed for local testing.

---

# 19. Outline API Request

The effective request structure was:

```http
POST /api/users.invite
Authorization: Bearer <OUTLINE_API_KEY>
Content-Type: application/json
```

with a body equivalent to:

```json
{
  "invites": [
    {
      "email": "dev873-test@example.com",
      "name": "DEV873 Test",
      "role": "member"
    }
  ],
  "suppressEmail": true
}
```

The actual API key is not included in this documentation.

---

# 20. Shape Response

After Outline processes the invitation request, the workflow passes the result through the Shape Response node.

The purpose of this node is to provide a consistent provisioning response regardless of the target application's native response structure.

The workflow returns the employee identity in the standardized response format used by the DEV-868 project.

---

# 21. Successful Provisioning Test

The final DEV-873 workflow was executed successfully.

The test request used the configured provisioning secret and the test employee:

```text
dev873-test@example.com
```

The Outline API returned:

```text
HTTP 201 Created
```

The successful n8n response was:

```json
{
  "username": null
}
```

The `username` value being `null` reflects the response mapping used for the Outline invitation operation.

The important result was that Outline accepted and created the invitation operation successfully.

---

# 22. Outline API Log Verification

The successful request was also visible in the Outline application logs.

The API request returned:

```text
201
```

confirming that the Outline backend successfully processed the invitation request.

This provided an additional verification layer beyond the n8n response.

---

# 23. Outline Application Verification

The invited employee was verified in the Outline application.

The account/invitation appeared with the expected pending state.

The test account was:

```text
dev873-test@example.com
```

The account was assigned the standard:

```text
member
```

role.

This confirmed that the target application received the provisioning request successfully.

---

# 24. Evidence

The repository contains the following DEV-873 evidence screenshot:

```text
evidence/DEV-873-Outline-invite-pending.png
```

The screenshot shows the Outline invitation in its pending state.

This provides visual evidence that the provisioning operation reached the target Outline application.

---

# 25. Workflow File

The completed n8n workflow is stored in:

```text
workflows/flowable-provisioning-outline.json
```

The workflow can be imported into a compatible local n8n environment and configured with the required local credentials and environment variables.

The workflow file does not contain real secrets.

---

# 26. Credential Handling

The Outline API credential was stored in n8n credentials.

The repository does not contain:

* Outline API keys
* Administrator passwords
* Provisioning secrets
* Database passwords
* JWTs
* SMTP credentials
* Docker environment secrets

The actual local `.env` file remains outside version control.

---

# 27. Git Security

The project uses `.gitignore` rules to prevent local secret files from being committed.

Sensitive local files such as:

```text
.env
*.env
```

are excluded.

Only safe configuration templates should be committed.

For example:

```text
.env.example
```

can contain placeholder values but must never contain real credentials.

---

# 28. Development Issue: Docker Networking

One of the main technical issues encountered during DEV-873 was communication between the Dockerized n8n instance and the locally exposed Outline application.

The initial conceptual mistake was treating `localhost` inside the n8n container as if it referred to the host machine.

Inside a container:

```text
localhost
```

refers to that container.

The working host address was:

```text
host.docker.internal
```

Therefore the final Outline API URL was:

```text
http://host.docker.internal:3001/api/users.invite
```

This successfully allowed n8n to communicate with Outline.

---

# 29. Development Issue: Local Email Delivery

The Outline invitation endpoint is designed to support email invitations.

The local development environment did not require external email delivery.

The workflow therefore used:

```json
"suppressEmail": true
```

This allowed the invitation operation to be tested directly in the Outline application without configuring an external mail provider.

---

# 30. Development Issue: API Credential Security

The Outline API key was required for the integration.

The credential was stored in n8n rather than being written directly into the workflow body.

This follows the broader DEV-868 principle of separating application credentials from workflow logic.

The API key used during development should be revoked and regenerated if it has ever been exposed outside the intended local environment.

No real API key is included in this repository.

---

# 31. Testing Strategy

DEV-873 was tested at multiple levels.

### Layer 1 — Workflow request

The n8n webhook accepted the provisioning request.

### Layer 2 — Security

The shared provisioning secret was checked before the target API request.

### Layer 3 — API integration

n8n successfully reached the Outline API.

### Layer 4 — API response

Outline returned:

```text
201 Created
```

### Layer 5 — Application verification

The invited employee appeared in Outline as a pending invitation.

This provided end-to-end verification of the provisioning workflow.

---

# 32. End-to-End Flow

The complete successful flow was:

```text
Employee
   |
   | Provisioning Request
   v
n8n Webhook
   |
   | Shared Secret Validation
   v
Check Secret
   |
   | Valid
   v
Outline API
   |
   | POST /api/users.invite
   v
Outline
   |
   | Invitation Created
   v
HTTP 201
   |
   v
Shape Response
   |
   v
Provisioning Response
```

---

# 33. Security Flow

The security path is:

```text
Incoming Request
       |
       v
Shared Secret Check
       |
   +---+---+
   |       |
 Valid   Invalid
   |       |
   v       v
Outline   HTTP 401
API
```

This ensures that only requests containing the correct provisioning secret can reach the Outline provisioning API.

---

# 34. Reusability

DEV-873 uses the same provisioning architecture established for the other DEV-868 integrations.

The reusable pattern is:

```text
Webhook
   ↓
Secret Validation
   ↓
Target Application API
   ↓
Response Transformation
   ↓
Success / Unauthorized
```

This makes the provisioning system easier to extend to additional internal applications.

---

# 35. Comparison With Other DEV-868 Integrations

DEV-873 follows the same core design used by the other completed integrations:

| Task    | Target     | Provisioning Method        |
| ------- | ---------- | -------------------------- |
| DEV-869 | TwentyCRM  | Invitation API             |
| DEV-870 | Mattermost | User API + Team Membership |
| DEV-871 | NextERP    | User REST API              |
| DEV-872 | Superset   | Login API + User API       |
| DEV-873 | Outline    | Invitation API             |

Each integration keeps the common authentication and response-handling pattern while adapting the target application API.

---

# 36. What DEV-873 Demonstrates

The implementation demonstrates practical experience with:

* n8n workflow automation
* REST API integration
* Docker
* Docker-to-host networking
* API authentication
* Bearer tokens
* Webhooks
* Shared-secret validation
* JSON request bodies
* HTTP status codes
* Local self-hosted applications
* PostgreSQL-backed applications
* Redis-backed applications
* Security-conscious credential handling
* End-to-end application verification

---

# 37. Lessons Learned

### 37.1 Container networking is different from host networking

A service available at `localhost` on the host is not automatically available at the same address from inside another Docker container.

The correct Docker-to-host address used here was:

```text
host.docker.internal
```

### 37.2 API credentials should be separated from workflow logic

Credentials should be stored in n8n's credential system instead of hardcoding them into workflow nodes.

### 37.3 Invitation APIs can differ from user-creation APIs

Outline uses an invitation endpoint rather than a simple direct user-creation endpoint for this provisioning operation.

Understanding the target application's API model is therefore important when designing automation.

### 37.4 Local email requirements can be avoided during development

Using:

```text
suppressEmail=true
```

allowed the invitation workflow to be tested without introducing unnecessary external email infrastructure.

### 37.5 Application verification is important

A successful HTTP response alone does not provide complete end-to-end confidence.

The pending invitation was also checked in the Outline UI.

---

# 38. Production Considerations

This implementation was built for a local development and portfolio environment.

A production deployment would require additional controls, including:

* HTTPS
* Secure secret management
* Secret rotation
* Restricted API credentials
* Production-grade database configuration
* Production Redis configuration
* Proper SMTP/email configuration
* Network segmentation
* Audit logging
* Monitoring
* Rate limiting
* Access control
* Backup and recovery procedures

The local configuration should not be treated as a production deployment template without additional hardening.

---

# 39. Repository Structure

The relevant DEV-873 files are:

```text
dev-868-provisioning/
│
├── workflows/
│   └── flowable-provisioning-outline.json
│
├── docs/
│   └── README-DEV-873-Outline.md
│
└── evidence/
    └── DEV-873-Outline-invite-pending.png
```

---

# 40. Final Result

DEV-873 successfully demonstrated automated employee invitation provisioning into a local Outline instance through n8n.

The final implementation:

* Receives a provisioning request through an n8n webhook.
* Validates the shared provisioning secret.
* Authenticates against the Outline API.
* Sends an employee invitation.
* Assigns the standard member role.
* Suppresses external email delivery for local testing.
* Successfully reaches Outline through Docker-to-host networking.
* Receives an HTTP 201 response.
* Returns a standardized provisioning response.
* Verifies the pending invitation in the Outline UI.

## Status

**DEV-873 — Outline Account Provisioning: COMPLETE ✅**

The implementation completes the fifth application integration in the DEV-868 employee account provisioning project.

**Completed integrations:**

* DEV-869 — TwentyCRM
* DEV-870 — Mattermost
* DEV-871 — NextERP
* DEV-872 — Superset
* DEV-873 — Outline

**DEV-874 — Documenso was not implemented and is intentionally outside the completed project scope.**
