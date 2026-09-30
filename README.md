# DEV-868 — Employee Account Provisioning Platform

A local, self-hosted employee account provisioning platform built with **n8n, Docker, REST APIs, and application-specific authentication**.

The project automates employee account onboarding across multiple internal applications from a single provisioning request.

## Project Overview

The goal of DEV-868 was to build a reusable provisioning workflow that receives an employee onboarding request, validates a shared secret, communicates with a target application's API, and returns a standardized response.

Instead of manually creating accounts in each application, the provisioning request is handled through n8n and routed to the appropriate application API.

### Core Architecture

```text
Employee Provisioning Request
            ↓
       n8n Webhook
            ↓
   Shared Secret Validation
            ↓
      Target Application API
            ↓
       Shape Response
            ↓
   Success / Unauthorized
```

## Completed Integrations

| DEV     | Application       | Provisioning Method           | Status          |
| ------- | ----------------- | ----------------------------- | --------------- |
| DEV-869 | TwentyCRM         | GraphQL-style API request     | Complete        |
| DEV-870 | Mattermost        | REST API                      | Complete        |
| DEV-871 | NextERP / ERPNext | REST API                      | Complete        |
| DEV-872 | Apache Superset   | REST API + JWT authentication | Complete        |
| DEV-873 | Outline           | REST API invitation           | Complete        |
| DEV-874 | Documenso         | —                             | Not implemented |

The project contains **five completed application integrations**.

DEV-874 was intentionally not included in the completed implementation because it was not finished.

## Objectives

The project was designed to demonstrate practical experience with:

* Workflow automation
* REST API integration
* Webhooks
* API authentication
* Shared-secret validation
* Docker-based local infrastructure
* Application provisioning
* Role assignment
* Least-privilege access
* Error handling
* JSON request/response processing
* Infrastructure troubleshooting
* Reusable automation patterns
* Technical documentation

## Technology Stack

* **n8n** — workflow automation
* **Docker** — local application infrastructure
* **REST APIs** — application integration
* **GraphQL-style API operations** — TwentyCRM
* **JWT authentication** — Apache Superset
* **Header/API-key authentication** — target applications
* **Linux / Bash** — environment management and troubleshooting
* **JSON** — workflow definitions and API payloads

## Local Environment

The project was developed as a local/self-hosted environment.

### n8n

```text
http://localhost:5678
```

### TwentyCRM

```text
http://localhost:3000
```

### Mattermost

```text
http://localhost:8065
```

### NextERP / ERPNext

```text
http://localhost:8080
```

### Apache Superset

```text
http://localhost:8088
```

### Outline

```text
http://localhost:3001
```

Application URLs are local development endpoints and are not intended to be publicly exposed.

## Security Foundation

A major part of the project was removing hardcoded provisioning secrets from the workflow design.

The provisioning workflow uses environment-based configuration:

```text
PROVISIONING_SECRET
PROVISIONING_SECRET_REQUIRED
```

n8n was configured to allow controlled access to required environment variables.

The provisioning secret is validated before the target application's API is called.

### Unauthorized Requests

Requests with an invalid provisioning secret return:

```http
401 Unauthorized
```

Example response:

```json
{
  "error": "Unauthorized"
}
```

This prevents unauthenticated provisioning requests from reaching the target applications.

## Credential Management

Application credentials were stored outside the workflow JSON whenever possible using n8n credentials or environment configuration.

Examples include:

* API keys
* API secrets
* Bot tokens
* JWT credentials
* Administrator credentials
* Provisioning secrets

### Important

Real secrets, passwords, tokens, and API keys must **never** be committed to Git.

The repository therefore uses:

```text
.env.example
```

for safe configuration documentation.

Actual credentials remain local.

## Reusable Workflow Pattern

The provisioning workflows follow a common pattern:

```text
Webhook
   ↓
Check Secret
   ↓
Create Account / Invite User
   ↓
Shape Response
   ↓
Respond Success
```

An invalid secret follows the unauthorized branch:

```text
Webhook
   ↓
Check Secret
   ↓
Unauthorized
   ↓
HTTP 401
```

This pattern makes the provisioning approach reusable across different applications while allowing each integration to use its own API and authentication mechanism.

# Application Integrations

## DEV-869 — TwentyCRM

TwentyCRM was integrated using its local API.

The workflow sends a GraphQL-style `sendInvitations` operation through the TwentyCRM metadata endpoint.

The workflow:

1. Receives the employee email.
2. Validates the provisioning secret.
3. Sends the invitation request to TwentyCRM.
4. Assigns the required Member role.
5. Returns a standardized provisioning response.

The integration also required configuring an API key in TwentyCRM and storing it securely in n8n.

### Result

The workflow successfully generated a provisioning response containing the requested username/email.

Detailed implementation:

```text
docs/README-DEV-869-TwentyCRM.md
```

---

## DEV-870 — Mattermost

Mattermost was deployed locally using Docker Compose.

The integration provisions a Mattermost user through the REST API and adds the user to the DEV-868 team.

The workflow:

1. Receives employee information.
2. Validates the shared secret.
3. Creates the Mattermost account.
4. Adds the account to the DEV-868 team.
5. Returns the provisioning response.

Mattermost required enabling user creation/open-server settings during local development.

### Result

The provisioned user was successfully verified inside Mattermost and associated with the DEV-868 team.

Detailed implementation:

```text
docs/README-DEV-870-Mattermost.md
```

---

## DEV-871 — NextERP / ERPNext

NextERP was deployed locally using the Frappe/ERPNext Docker environment.

The provisioning integration uses the ERPNext REST API.

A dedicated provisioning API user was created and authenticated using an API key and secret.

The workflow:

1. Receives employee information.
2. Validates the shared secret.
3. Authenticates using the provisioning API credentials.
4. Creates an ERPNext System User.
5. Assigns the standard Employee role.
6. Returns the provisioning response.

The provisioning account was intentionally kept separate from elevated administrator roles.

### Result

The provisioned employee account was successfully created and verified in ERPNext with the required Employee role.

Detailed implementation:

```text
docs/README-DEV-871-NextERP.md
```

---

## DEV-872 — Apache Superset

Apache Superset was deployed locally and configured to expose the required security API.

Superset uses JWT-based authentication for the provisioning API.

The workflow performs two API operations:

```text
Superset Login
      ↓
JWT Token
      ↓
Create User
```

The workflow first authenticates against:

```text
/api/v1/security/login
```

The returned JWT is then used as a Bearer token when creating the employee account.

The new account is assigned the **Gamma** role.

### Result

The workflow successfully created a Superset user and the account was verified through:

```text
Settings → List Users
```

The provisioned account was confirmed with the Gamma role.

Detailed implementation:

```text
docs/README-DEV-872-Superset.md
```

---

## DEV-873 — Outline

Outline was deployed locally and integrated through its API.

Unlike applications that directly create a password-based account, Outline uses an invitation-based onboarding model.

The workflow therefore creates a pending member invitation.

The workflow:

1. Receives employee information.
2. Validates the shared secret.
3. Calls the Outline invitation endpoint.
4. Creates a member invitation.
5. Returns the provisioning response.

The invitation request uses the member role.

### Result

The provisioning request successfully returned HTTP 201 and the invitation was verified in the Outline interface as pending.

This demonstrates an important distinction between direct account creation and invitation-based provisioning.

Detailed implementation:

```text
docs/README-DEV-873-Outline.md
```

# Provisioning Request Model

The integrations use a common employee provisioning concept.

Example:

```json
{
  "employeeName": "Jane Doe",
  "employeeEmail": "jane.doe@company.com",
  "password": "temporary-password"
}
```

The exact fields used by each application depend on the application's API and onboarding model.

Passwords and other sensitive values must never be hardcoded into committed workflow files.

# Standard Response

Successful workflows return a simplified response such as:

```json
{
  "username": "employee@example.com"
}
```

This provides a consistent interface to the provisioning caller while hiding application-specific API response structures.

Unauthorized requests return:

```json
{
  "error": "Unauthorized"
}
```

with HTTP status:

```text
401
```

# Docker Networking

A recurring issue during local development was the difference between the host machine and the n8n Docker container.

When an application was running on the host machine, using:

```text
localhost
```

from inside the n8n container did not always point to the host.

The solution was to use:

```text
host.docker.internal
```

for host-accessible services.

Example:

```text
http://host.docker.internal:8088
```

This was particularly important for the Apache Superset integration.

# Testing Strategy

Each completed integration was tested through the actual n8n workflow.

Testing focused on:

### Successful Provisioning

A valid provisioning secret should:

1. Pass secret validation.
2. Reach the target application API.
3. Create/invite the requested account.
4. Return a successful response.

### Invalid Secret

An invalid provisioning secret should:

1. Fail validation.
2. Prevent the target API request.
3. Return HTTP 401.
4. Return:

```json
{
  "error": "Unauthorized"
}
```

### Target Application Verification

Where applicable, the created account was verified directly inside the target application's interface.

This ensured that successful HTTP execution corresponded to an actual provisioning result rather than merely a successful workflow execution.

# Evidence

Evidence was captured for the completed integrations.

The repository structure reserves:

```text
evidence/
```

for screenshots demonstrating successful provisioning and target-application results.

Examples include:

```text
DEV-869-twentycrm-successful-execution.png
DEV-870-mattermost-successful-execution.png
DEV-870-mattermost-provisioned-user.png
DEV-871-NextERP-successful-execution.png
DEV-871-NextERP-user-provisioned.png
DEV-872-n8n-successful-provisioning.png
DEV-872-Superset-Gamma-user.png
DEV-873-Outline-invite-pending.png
```

Screenshots containing secrets, tokens, passwords, or other sensitive information should not be committed.

# Troubleshooting Lessons

Several practical infrastructure and API integration issues were encountered during development.

## 1. Docker-to-host connectivity

`localhost` inside a container does not necessarily refer to the host machine.

For host services:

```text
host.docker.internal
```

was used.

## 2. n8n expressions

n8n expression syntax needs to match the field's expression mode.

For example:

```text
{{ $json.body.employeeEmail }}
```

was used instead of unnecessarily adding an extra leading `=`.

## 3. API authentication

Different applications required different authentication mechanisms:

* API keys
* Header authentication
* Bearer tokens
* JWT
* API key + API secret

The workflow was adapted to the authentication model of each target application.

## 4. Application-specific provisioning models

Not every application provisions users in the same way.

Examples:

* TwentyCRM — invitation-based API operation
* Mattermost — direct user creation + team membership
* ERPNext — direct user creation + role assignment
* Superset — authenticated user creation + role assignment
* Outline — invitation-based member onboarding

The project therefore demonstrates integration flexibility rather than assuming a single universal provisioning API.

# Security Lessons

The project reinforced several important security practices:

* Do not hardcode secrets in workflow definitions.
* Store credentials separately from application logic.
* Validate requests before calling target APIs.
* Return unauthorized responses for invalid secrets.
* Use dedicated provisioning identities where possible.
* Avoid unnecessary administrator privileges.
* Use least-privilege application roles.
* Never commit `.env` files.
* Never commit API keys or bot tokens.
* Rotate credentials that become exposed during development.
* Review exported workflow JSON before publishing it.

# GitHub Security Checklist

Before pushing the project publicly:

```text
[ ] No real provisioning secret
[ ] No API keys
[ ] No API secrets
[ ] No bot tokens
[ ] No JWTs
[ ] No administrator passwords
[ ] No temporary passwords
[ ] No private credentials inside workflow JSON
[ ] .env is ignored
[ ] .env.example contains placeholders only
[ ] Screenshots contain no sensitive credentials
```

# Suggested Repository Structure

```text
dev-868-provisioning/
│
├── README.md
│
├── workflows/
│   ├── flowable-provisioning-twentycrm.json
│   ├── flowable-provisioning-superset.json
│   └── flowable-provisioning-outline.json
│
├── docs/
│   ├── README-DEV-869-TwentyCRM.md
│   ├── README-DEV-870-Mattermost.md
│   ├── README-DEV-871-NextERP.md
│   ├── README-DEV-872-Superset.md
│   └── README-DEV-873-Outline.md
│
├── evidence/
│   ├── DEV-869-twentycrm-successful-execution.png
│   ├── DEV-870-mattermost-successful-execution.png
│   ├── DEV-870-mattermost-provisioned-user.png
│   ├── DEV-871-NextERP-successful-execution.png
│   ├── DEV-871-NextERP-user-provisioned.png
│   ├── DEV-872-n8n-successful-provisioning.png
│   ├── DEV-872-Superset-Gamma-user.png
│   └── DEV-873-Outline-invite-pending.png
│
├── .gitignore
│
└── .env.example
```

# Project Outcome

DEV-868 demonstrates a complete local employee provisioning automation pattern across five different applications.

The implementation covers:

* Webhook-driven provisioning
* Shared-secret request validation
* REST API integration
* GraphQL-style API integration
* JWT authentication
* API-key authentication
* Role assignment
* Team membership
* Invitation-based onboarding
* Docker infrastructure
* Error handling
* Least-privilege provisioning
* Reusable n8n workflows
* Technical documentation
* Provisioning verification

## Final Status

| Task    | Application       | Status            |
| ------- | ----------------- | ----------------- |
| DEV-869 | TwentyCRM         | ✅ Complete        |
| DEV-870 | Mattermost        | ✅ Complete        |
| DEV-871 | NextERP / ERPNext | ✅ Complete        |
| DEV-872 | Apache Superset   | ✅ Complete        |
| DEV-873 | Outline           | ✅ Complete        |
| DEV-874 | Documenso         | ⏸ Not implemented |

### Final Architecture

```text
                    Employee Request
                           │
                           ▼
                    ┌─────────────┐
                    │     n8n     │
                    │   Webhook   │
                    └──────┬──────┘
                           │
                           ▼
                  Shared Secret Check
                           │
             ┌─────────────┴─────────────┐
             │                           │
          Valid                       Invalid
             │                           │
             ▼                           ▼
     Target Application API       HTTP 401 Unauthorized
             │
      ┌──────┼────────┬──────────┐
      │      │        │          │
      ▼      ▼        ▼          ▼
   Twenty  Mattermost ERPNext  Superset
      │      │        │          │
      └──────┴────────┴──────────┘
                    │
                    ▼
                  Outline
                    │
                    ▼
              Standard Response
```

## What This Project Demonstrates

This project demonstrates practical ability to design and implement automation around real application APIs rather than simply configuring an isolated automation workflow.

It combines:

```text
Linux
Docker
n8n
REST APIs
Authentication
Webhooks
JSON
Application provisioning
Role-based access control
Troubleshooting
Documentation
```

The result is a reproducible local provisioning platform that can serve as a foundation for extending employee onboarding automation to additional internal applications.

## Final Takeaway

DEV-868 demonstrates how a centralized automation layer can reduce repetitive employee onboarding work while enforcing authentication, application-specific authorization, and consistent provisioning behavior.

The project was intentionally built locally with Docker and n8n so that the complete provisioning flow could be developed, tested, debugged, and documented without depending on external production infrastructure.
