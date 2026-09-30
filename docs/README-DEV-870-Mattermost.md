# DEV-870 — Mattermost Account Provisioning

## 1. Overview

DEV-870 implements automated employee account provisioning for a local **Mattermost** instance using **n8n**.

The workflow receives an employee provisioning request, validates a shared secret, creates the Mattermost user through the REST API, adds the user to the DEV-868 team, and returns a standardized response.

### Architecture

```text
Employee Provisioning Request
          ↓
     n8n Webhook
          ↓
   Shared Secret Check
          ↓
   Mattermost User API
          ↓
 Add User to DEV-868 Team
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

### Mattermost

Working directory:

```bash
~/mattermost-local
```

Application URL:

```text
http://localhost:8065
```

Mattermost version:

```text
11.7.0
```

---

# 3. Mattermost Setup

Mattermost was deployed using the official Docker Compose configuration.

Initial permission problems occurred with the mounted application directories.

They were corrected with:

```bash
sudo chown -R 2000:2000 \
  volumes/app/mattermost/config \
  volumes/app/mattermost/data \
  volumes/app/mattermost/logs \
  volumes/app/mattermost/plugins
```

Mattermost health was verified with:

```bash
curl -i http://localhost:8065/api/v4/system/ping
```

The API returned HTTP 200.

---

# 4. Mattermost Provisioning Bot

A dedicated bot was created for n8n provisioning.

### Bot

Username:

```text
n8n-provisioner
```

Display name:

```text
n8n Provisioner
```

Description:

```text
DEV-868 employee provisioning
```

The bot was added to the:

```text
DEV-868
```

team.

Team ID:

```text
uztwn1j5zpn5zxmijq4ykcsjph
```

The bot authentication token was stored in an n8n Header Auth credential.

### Security note

The bot token was accidentally exposed during development.

It should be revoked and regenerated as a post-project security cleanup.

The replacement token should be stored securely and never committed to Git or documentation.

---

# 5. Mattermost API User Creation

Mattermost initially rejected API user creation with:

```text
api.user.create_user.no_open_server
```

The following settings were enabled:

```text
TeamSettings.EnableOpenServer = true
TeamSettings.EnableUserCreation = true
```

After restarting Mattermost, API user creation succeeded with HTTP 201.

The newly created user was also successfully added to the DEV-868 team.

---

# 6. Shared Security

The provisioning workflow uses the common DEV-868 environment-based secret:

```bash
export PROVISIONING_SECRET="dev868-test-secret-2026"
export PROVISIONING_SECRET_REQUIRED="true"
```

The request header is:

```text
x-provisioning-secret
```

Invalid secrets return:

```text
HTTP 401 Unauthorized
```

The secret is not hardcoded into the workflow.

---

# 7. n8n Workflow

The final DEV-870 workflow is:

```text
Webhook
    ↓
Check Secret
    ↓
Create Account
    ↓
Add User to DEV-868 Team
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

# 8. Webhook

The test webhook URL was:

```text
http://localhost:5678/webhook-test/d7a7475c-2c2a-45b9-aca1-ad92c22c0592
```

Example request body:

```json
{
  "employeeEmail": "mattermost-final@example.com",
  "username": "mattermostfinal",
  "password": "TestPass!2026"
}
```

The shared secret is supplied using:

```text
x-provisioning-secret
```

---

# 9. Create Account Node

### Method

```text
POST
```

### URL

```text
http://host.docker.internal:8065/api/v4/users
```

### Authentication

```text
Generic Credential Type
→ Header Auth
```

Header:

```text
Authorization: Bearer <Mattermost bot token>
```

### JSON body

```json
{
  "email": "{{ $json.body.employeeEmail }}",
  "username": "{{ $json.body.username.toLowerCase() }}",
  "password": "{{ $json.body.password }}"
}
```

The expressions intentionally do not contain a leading `=`.

---

# 10. Add User to DEV-868 Team

After creating the user, the workflow adds the new user to the DEV-868 team.

### Method

```text
POST
```

### URL

```text
http://host.docker.internal:8065/api/v4/teams/uztwn1j5zpn5zxmijq4ykcsjph/members
```

### JSON body

```json
{
  "team_id": "uztwn1j5zpn5zxmijq4ykcsjph",
  "user_id": "{{ $json.id }}"
}
```

The API returned a successful team membership object.

This confirmed that the provisioned employee was added to the correct team.

---

# 11. Shape Response

The Shape Response node normalizes the workflow output.

Mapping:

```text
username
=
{{ $('Webhook').item.json.body.employeeEmail }}
```

---

# 12. Success Response

The workflow returns:

```json
{
  "username": "{{ $json.username }}"
}
```

---

# 13. End-to-End Verification

The final workflow test returned:

```json
{
  "username": "mattermost-final@example.com"
}
```

This confirmed the complete chain:

```text
Webhook
→ Shared-secret validation
→ Mattermost user creation
→ DEV-868 team membership
→ Response shaping
→ Success response
```

---

# 14. Provisioned Account

The successfully provisioned Mattermost account was:

```text
Email: mattermost-final@example.com
Username: mattermostfinal
Team: DEV-868
```

The account was successfully visible in the Mattermost interface.

---

# 15. Docker Networking Lesson

Because n8n runs inside Docker while Mattermost is exposed on the host, the workflow accesses Mattermost through:

```text
http://host.docker.internal:8065
```

rather than:

```text
http://localhost:8065
```

Inside the n8n container, `localhost` refers to the n8n container itself.

---

# 16. n8n Test Webhook Lesson

Test webhooks require the workflow to be actively listening.

Correct sequence:

```text
Click Execute Workflow
        ↓
Run curl once
        ↓
Execution completes
```

The green node indicators represent the current execution.

Refreshing the workflow canvas removes those temporary execution indicators.

Therefore, execution screenshots should be taken immediately after a successful run without refreshing the workflow.

---

# 17. Evidence

The following screenshots were captured:

```text
DEV-870-mattermost-successful-execution.png
DEV-870-mattermost-provisioned-user.png
```

### Successful execution

`DEV-870-mattermost-successful-execution.png`

Shows the successful n8n execution path.

### Provisioned account

`DEV-870-mattermost-provisioned-user.png`

Shows the actual provisioned Mattermost account and its DEV-868 team membership.

---

# 18. Troubleshooting Lessons

### Mattermost filesystem permissions

Mattermost's mounted directories must be writable by the Mattermost container user.

The required ownership was:

```text
UID 2000
```

### Mattermost user creation

The API initially rejected user creation because open user creation was disabled.

The required settings were:

```text
EnableOpenServer = true
EnableUserCreation = true
```

### n8n expressions

Correct:

```text
{{ $json.body.username }}
```

Incorrect:

```text
={{ $json.body.username }}
```

An unintended leading `=` can cause invalid values to be sent to the target API.

### Docker networking

Use:

```text
host.docker.internal
```

when n8n needs to reach an application exposed on the Docker host.

---

# 19. Result

DEV-870 successfully demonstrated automated Mattermost employee provisioning through n8n using:

* Webhook-based provisioning
* Shared-secret validation
* Mattermost REST API
* Header-based API authentication
* Automated team membership
* Standardized response handling
* Unauthorized request handling
* Docker-to-host networking

## Status

**DEV-870 — Mattermost: Complete**
