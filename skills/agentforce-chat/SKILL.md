# Agentforce Chat Skill

Start and manage an interactive conversation with any Agentforce agent via the Einstein AI Agent Runtime API.

---

## When to invoke

Trigger this skill when the user wants to:
- Start a conversation / chat session with an Agentforce agent
- Ask questions to a specific agent by name
- Test an agent interactively
- Send a message to an agent and see its response

---

## Workflow

### Step 1 — Identify the target org and agent

#### Credential lookup

Check project files for credentials before asking the user. An org may have two Connected Apps — use the right one:
- **Data Cloud Connected App** (often in `.mcp.json`) — scope includes `cdp_query_api`. Do NOT use for agent API calls.
- **Agentforce Connected App** (e.g. "Headless Agent API") — scope includes `chatbot_api`. Use this one.

If credentials are not found, prompt for Instance URL, Consumer Key, and Consumer Secret. Resolve the agent name via `BotDefinition` — never ask the user for the agent ID:
```sql
SELECT Id, DeveloperName, MasterLabel FROM BotDefinition
WHERE MasterLabel LIKE '%<name>%' OR DeveloperName LIKE '%<name>%'
```

After a successful session with new credentials, offer to save them to `.connected-app-credentials.json` and remind the user to add it to `.gitignore`.

**Credentials file format:**
```json
[
  {
    "label": "my-org / Headless Agent API / AgentName",
    "instanceUrl": "https://myorg.salesforce.com",
    "clientId": "...",
    "clientSecret": "...",
    "agentId": "0Xx...",
    "apiHost": "https://api.gov.salesforce.com"
  }
]
```


### Step 2 — Get OAuth access token

```bash
curl -s -X POST "https://<instanceUrl>/services/oauth2/token" \
  -d "grant_type=client_credentials" \
  -d "client_id=<consumerKey>" \
  -d "client_secret=<consumerSecret>"
```

The response contains `access_token` and `api_instance_url`. Store the token.

**Do NOT use `api_instance_url` from the response as the API base** — it returns `https://api.salesforce.com` even for Gov Cloud orgs where it is incorrect.

Determine the correct API host based on org type:
- **Gov Cloud orgs** → `https://api.gov.salesforce.com`
- **Commercial orgs** → `https://api.salesforce.com`

Use this API host for ALL subsequent calls (session create, message send, session end).

### Step 3 — Create a session

```bash
curl -s -X POST "<apiHost>/einstein/ai-agent/v1/agents/<agentId>/sessions" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "externalSessionKey": "<uuid>",
    "instanceConfig": {"endpoint": "<instanceUrl>"},
    "tz": "America/Los_Angeles",
    "variables": [
      {"name": "$Context.EndUserLanguage", "type": "Text", "value": "en_US"}
    ],
    "featureSupport": "Streaming",
    "streamingCapabilities": {"chunkTypes": ["Text"]},
    "bypassUser": true
  }'
```

The response contains `sessionId` and the agent's opening message. Display the opening message to the user.

**Important:** The response also contains `_links.messages.href` — **ignore it**. It may point to the wrong host (e.g. `api.salesforce.com` for a Gov Cloud org). Always construct the message URL manually using the correct `<apiHost>`.

### Step 4 — Send messages

For each user message, increment the `sequenceId`. Always use `<apiHost>` (not `_links` href):

```bash
curl -s -X POST "<apiHost>/einstein/ai-agent/v1/sessions/<sessionId>/messages" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"message": {"sequenceId": <n>, "type": "Text", "text": "<userMessage>"}}'
```

Extract and display the agent's response from `messages[].message` in the response.

**Always show the agent's raw response exactly as returned — no paraphrasing, no editing.**

### Step 5 — End the session

When the user is done or says "end session" / "bye" / "close":

```bash
curl -s -X DELETE "<apiHost>/einstein/ai-agent/v1/sessions/<sessionId>" \
  -H "Authorization: Bearer <token>"
```

---

## Key technical notes

- Message type must be `"Text"` (not `"Human"`)
- Token expires — refresh using client credentials if a 401 is returned mid-session
- **Gov Cloud orgs use `https://api.gov.salesforce.com`** — `api_instance_url` in the OAuth response returns `api.salesforce.com` even for Gov Cloud; do not trust it
- **Never use `_links.messages.href`** from the session response — always construct the URL manually with the correct API host
- Session IDs follow the format `019e....`
- The Connected App must have `chatbot_api` in its OAuth scopes — use the **Agentforce Connected App** (e.g. "Headless Agent API"), not the Data Cloud one
- Use Python `urllib` for API calls when the token is a long JWT — shell variable assignment truncates tokens over ~100 chars

---

## Error handling

| Error | Fix |
|---|---|
| `INVALID_JWT_FORMAT` | Token format wrong — use the JWT from client credentials, not the SF session token |
| `Invalid token` | Token expired — re-run OAuth to get a fresh token |
| `410 Gone` on messages | Session expired — create a new session |
| `404` on session create | Wrong API host — check if org is Gov Cloud (`api.gov.salesforce.com`) vs commercial (`api.salesforce.com`) |
| `404` on message send | Wrong API host — do not use `_links` href; construct URL manually with correct host |
| `type: Human not found` | Use `"type": "Text"` not `"type": "Human"` |
| Token truncated in shell | Use Python `urllib` instead of `curl` + shell variable for long JWT tokens |
