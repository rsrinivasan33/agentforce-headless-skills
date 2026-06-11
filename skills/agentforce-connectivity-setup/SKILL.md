# Agentforce Connectivity Setup Skill

Guide the user through setting up and verifying connectivity to an Agentforce agent via the Einstein AI Agent Runtime API.

---

## When to invoke

Trigger this skill when the user wants to:
- Connect to an Agentforce agent for the first time
- Verify a Connected App is properly configured for agent API access
- Troubleshoot why they can't reach an agent
- Set up the credentials needed to chat with or test an agent

---

## Workflow

### Step 1 — Confirm the Salesforce org

Run `sf org list` and display the authenticated orgs. Ask the user which org to use if not already established.

### Step 2 — Verify the agent exists

Query BotDefinition using the org's session token:

```bash
curl -s "https://<instanceUrl>/services/data/v67.0/query?q=SELECT+Id,DeveloperName,MasterLabel,Description,Type,AgentType+FROM+BotDefinition+WHERE+Id='<agentId>'" \
  -H "Authorization: Bearer <sessionToken>"
```

If the agent ID is unknown, list all agents:
```bash
SELECT Id, DeveloperName, MasterLabel, Type FROM BotDefinition ORDER BY MasterLabel
```

Display the agent's name, type, and description so the user can confirm it's the right one.

### Step 3 — Identify and verify the Agentforce Connected App

**An org may have two Connected Apps — use the right one:**
- **Data Cloud Connected App** (often in `.mcp.json`) — scopes include `cdp_query_api`. Do NOT use for agent API calls.
- **Agentforce Connected App** (e.g. "Headless Agent API") — scopes include `chatbot_api`. Use this one.

Check project files for credentials before asking the user. If not found, ask for the **Consumer Key** and **Consumer Secret** from their Agentforce Connected App (Setup → App Manager → "Headless Agent API" or equivalent → Manage Consumer Details).

The Connected App **must** have these OAuth scopes:
- `chatbot_api` — required for agent conversation API
- `sfap_api` — required for Einstein AI services
- `api` — required for general API access

Test the token:
```bash
curl -s -X POST "https://<instanceUrl>/services/oauth2/token" \
  -d "grant_type=client_credentials" \
  -d "client_id=<consumerKey>" \
  -d "client_secret=<consumerSecret>"
```

Check that the returned token's `scope` field includes `chatbot_api`. If not, the Connected App needs to be updated in Setup.

### Step 4 — Determine API host and test session creation

**Do NOT use `api_instance_url` from the OAuth response** — it returns `api.salesforce.com` even for Gov Cloud orgs where it is incorrect.

Determine the correct API host:
- **Gov Cloud orgs** → `https://api.gov.salesforce.com`
- **Commercial orgs** → `https://api.salesforce.com`

Create a test session with this body:

```json
{
  "externalSessionKey": "<uuid>",
  "instanceConfig": {"endpoint": "<instanceUrl>"},
  "tz": "America/Los_Angeles",
  "variables": [{"name": "$Context.EndUserLanguage", "type": "Text", "value": "en_US"}],
  "featureSupport": "Streaming",
  "streamingCapabilities": {"chunkTypes": ["Text"]},
  "bypassUser": true
}
```

Use Python `urllib` if the token is a long JWT — shell variables truncate it. A successful response returns `sessionId` and an opening message from the agent. Display both.

### Step 5 — Clean up test session

```bash
curl -s -X DELETE "<apiHost>/einstein/ai-agent/v1/sessions/<sessionId>" \
  -H "Authorization: Bearer <token>"
```

### Step 6 — Report and save credentials summary

Display a connectivity summary:

```
✓ Org:              <alias> (<username>)
✓ Agent:            <MasterLabel> (<Id>)
✓ Agent Type:       <AgentType>
✓ Connected App:    <appName> (Agentforce)
✓ OAuth Scopes:     chatbot_api, sfap_api, api
✓ API Host:         <apiHost>  (gov or commercial)
✓ Session test:     PASSED — agent responded
```

Offer to proceed directly to `/agentforce-chat` or `/agentforce-testing`.

---

## Key technical notes

- **Gov Cloud orgs use `https://api.gov.salesforce.com`** — `api_instance_url` in the OAuth response is wrong for Gov Cloud; do not trust it
- `client_credentials` grant type requires "Enable Client Credentials Flow" checked on the Connected App in Setup
- Use the **Agentforce Connected App** (e.g. "Headless Agent API"), not the Data Cloud Connected App
- Use Python `urllib` for API calls when token is a long JWT — shell variables truncate it

---

## Common setup issues

| Symptom | Cause | Fix |
|---|---|---|
| `invalid_client` on token request | Wrong Consumer Key/Secret or wrong Connected App | Confirm you're using the Agentforce app, not Data Cloud |
| `chatbot_api` missing from token scope | Using Data Cloud Connected App instead of Agentforce | Use the "Headless Agent API" Connected App |
| `404` on session create | Wrong API host for Gov Cloud | Use `api.gov.salesforce.com` for Gov Cloud orgs |
| `404` on message send | Using `_links` href which points to wrong host | Construct URL manually using the correct API host |
| `INVALID_JWT_FORMAT` | Using SF session token instead of OAuth token | Must use client credentials token |
| `Invalid token` on messages | Token expired | Re-run OAuth to get a fresh token |
