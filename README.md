# Agentforce Headless API — Claude Code Skills

Three Claude Code skills for connecting to, chatting with, and testing any Agentforce agent via the Einstein AI Agent Runtime API — without the Salesforce UI.

Works on **any org, including Gov Cloud**, where the native Agentforce testing suite is not available.

---

## Skills

| Skill | Purpose |
|---|---|
| `agentforce-connectivity-setup` | First-time setup — verify Connected App, API host, and create a test session |
| `agentforce-chat` | Interactive conversation with an agent from Claude Code |
| `agentforce-testing` | Batch QA testing — send questions from Excel, capture responses, score Pass/Fail |

---

## Prerequisites

| Requirement | Details |
|---|---|
| Claude Code | CLI installed (`claude --version` to verify) |
| Salesforce CLI (`sf`) | v2.x or later (`sf --version` to verify) |
| Python 3 | Available on PATH (`python3 --version` to verify) |
| Agentforce org | Any org with an Agentforce agent deployed |
| Connected App | With `chatbot_api` + Client Credentials Flow enabled (see below) |

---

## Install

```bash
# Clone this repo
git clone https://github.com/<your-org>/agentforce-headless-skills.git

# Copy skills to your Claude Code skills directory
cp -r agentforce-headless-skills/skills/* ~/.claude/skills/
```

Then start Claude Code in your project directory and invoke any skill with `/agentforce-connectivity-setup`, `/agentforce-chat`, or `/agentforce-testing`.

---

## First-Time Setup

### 1. Create an Agentforce Connected App

In your Salesforce org: **Setup → App Manager → New Connected App**

Required OAuth scopes:
- `Access Chatbot services (chatbot_api)`
- `Access Einstein AI services (sfap_api)`
- `Manage user data via APIs (api)`

Check **Enable Client Credentials Flow**, then **Edit Policies → Client Credentials Flow → Run As** — set a Run As user. Without this, auth will fail.

### 2. Get your Consumer Key and Secret

App Manager → find your Connected App → **View → Manage Consumer Details**

### 3. Find your Agent ID

Run in Developer Console or Workbench:
```sql
SELECT Id, DeveloperName, MasterLabel FROM BotDefinition
```
The `Id` (starts with `0Xx`) is your agent ID.

### 4. Create a credentials file

In your project root, create `.connected-app-credentials.json`:

```json
[
  {
    "label": "my-org / Headless Agent API / My Agent",
    "instanceUrl": "https://<your-org>.my.salesforce.com",
    "clientId": "<Consumer Key>",
    "clientSecret": "<Consumer Secret>",
    "agentId": "0Xx...",
    "apiHost": "https://api.salesforce.com"
  }
]
```

For Gov Cloud orgs, set `apiHost` to `https://api.gov.salesforce.com`.

Add this file to `.gitignore` — it contains secrets.

### 5. Run setup

```
/agentforce-connectivity-setup
```

This verifies the Connected App scopes, API host, and creates a test session to confirm the full chain works before you use chat or testing.

---

## API Host — Gov Cloud vs Commercial

| Org Type | API Host |
|---|---|
| Gov Cloud | `https://api.gov.salesforce.com` |
| Commercial | `https://api.salesforce.com` |

The OAuth token response contains `api_instance_url` — **do not use it**. For Gov Cloud orgs it returns `api.salesforce.com`, which is wrong. The skills handle this automatically when `apiHost` is set correctly in the credentials file.

---

## Why Headless API?

The native Salesforce Agentforce testing suite is not available on Gov Cloud orgs. Teams on Gov Cloud — federal agencies, regulated industries, public sector — have no out-of-the-box way to run systematic test cases against a live agent.

This approach communicates with the agent directly via the Einstein AI Agent Runtime API, bypassing the platform UI entirely. It works on any org.

| Who uses it | What they get |
|---|---|
| QA testers | Run a spreadsheet of questions, get responses written back — no Salesforce login needed |
| Product owners | Expected answers + regression scoring (Pass / Fail / Review) after every agent change |
| Developers | Spot-check agent behavior from the terminal immediately after a code change |
| Business stakeholders | Contribute questions in Excel; review responses in the same file |

---

## Security Notes

- **Never commit `.connected-app-credentials.json`** — add it to `.gitignore`
- The skill files in this repo contain no credentials, no org-specific references, and are safe to share publicly
- Each user creates their own credentials file locally
