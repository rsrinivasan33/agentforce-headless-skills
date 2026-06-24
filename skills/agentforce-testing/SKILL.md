# Agentforce Testing Skill

Test an Agentforce agent — either for specific changes/topics or a full regression sweep — and capture responses for human review.

---

## When to invoke

Trigger this skill when the user wants to:
- Test an Agentforce agent after a change
- Run QA test cases against an Agentforce agent
- Do regression testing on an agent
- Test an agent for specific topics or scenarios
- Capture agent responses into a test results file
- Automate batch testing of an agent

---

## Workflow

### Step 1 — Identify the agent

Ask: **What is the name of the agent you want to test?**

Query `BotDefinition` to resolve the name:
```sql
SELECT Id, DeveloperName, MasterLabel, Description, AgentType
FROM BotDefinition
WHERE MasterLabel LIKE '%<name>%' OR DeveloperName LIKE '%<name>%'
```

Show the match and confirm with the user before proceeding. If multiple matches, list them and ask the user to pick one.

### Step 2 — Confirm the org

Run `sf org list` if not already confirmed this session. Ask which org to use and confirm the agent ID for that specific org. Agent IDs are org-specific — always query `BotDefinition` by name to get the correct ID for the target org.

### Step 3 — Testing scope

Ask: **Are you testing for specific changes/topics, or doing a full regression?**

---

### Path A — Specific changes/topics

**Step A1 — Get context**

Ask: What changed, or which topics do you want to cover? Get enough detail to generate meaningful, grounded questions.

**Step A2 — Question source**

Ask: Do you have a question file, or should I generate the questions?
- **Local file** → ask for file path, sheet name, and question column
- **Salesforce Data Library** → ask for the file name/reference
- **Generate** → proceed to Step A3

**Step A3 — Generate questions (only if generating)**

Ask: How many questions do you want?

Generate questions in three categories based on the context the user provided:
- **Direct topic questions** — straightforward questions about the change/topic
- **Edge cases** — tricky or boundary variations of the topic (e.g. refinancing, eligibility edge cases, unusual scenarios)
- **Out-of-scope / deflection** — questions the agent should refuse or redirect, to verify it stays in its lane

Show the generated questions to the user and ask for confirmation before running. Make adjustments if the user requests them.

Save the confirmed questions to `questions_<agentName>_<date>.xlsx`.

---

### Path B — Full regression

**Step B1 — Question file**

Ask: Do you have the question file locally or in Salesforce Data Library?
- Ask for file path/reference, sheet name, and question column

**Step B2 — Expected answers**

Ask: Does the file have a column with expected or acceptable answers?
- **Yes** → ask which column contains the expected answers → Claude will compare after running (see Step 7)
- **No** → skip comparison, responses only

---

### Step 4 — Multi-turn conversation mode

Before running, ask the user:

> "If the agent asks for more details mid-conversation (e.g. asks for a date, reservation ID, or other specifics), should I automatically generate plausible follow-up details and continue the exchange? Or should I record the clarifying question as the response and move on?"

- **Continue with generated details** — Claude fabricates realistic follow-up details (e.g. a sample date, a fake reservation ID) and sends them automatically. The full exchange is captured in the response cell.
- **Record and move on** — the agent's clarifying question is written as the response and Claude moves to the next question.

Store the user's choice and apply it consistently throughout the run.

If **continue with generated details** is chosen:
- When a response looks like a clarifying question (e.g. ends with "?", contains "please provide", "could you share"), generate a plausible follow-up and send it as the next message in the same session
- Capture the full conversation thread (question → agent clarifies → follow-up → agent final response) in the response cell, clearly labeled by turn
- Limit to **2 follow-up turns** per question to avoid infinite loops — if still unresolved, write what was captured

### Step 5 — Confirm output column

Inspect the file structure to understand its columns and sheets.

- Place the response column immediately after the last used column
- Name it: `Agent Response (<today's date>)`
- If comparison is enabled, reserve the next column for `Pass/Fail (<today's date>)`
- Confirm placement with the user if there is any ambiguity

### Step 6 — Get OAuth token and create session

#### Credential lookup

Check project files for credentials before asking the user. An org may have two Connected Apps — use the right one:
- **Data Cloud Connected App** (often in `.mcp.json`) — scopes include `cdp_query_api`. Do NOT use for agent API.
- **Agentforce Connected App** (e.g. "Headless Agent API") — scopes include `chatbot_api`. Use this one.

If not found, prompt for Instance URL, Consumer Key, and Consumer Secret. After a successful run, offer to save credentials to `.connected-app-credentials.json` and remind the user to add it to `.gitignore`.

**Credentials file format:**
```json
[
  {
    "label": "my-org / Headless Agent API",
    "instanceUrl": "https://myorg.salesforce.com",
    "clientId": "...",
    "clientSecret": "...",
    "apiHost": "https://api.gov.salesforce.com"
  }
]
```

Determine the correct API host — **do NOT use `api_instance_url` from the OAuth response**:
- **Gov Cloud orgs** → `https://api.gov.salesforce.com`
- **Commercial orgs** → `https://api.salesforce.com`

Use Python `urllib` for all API calls to avoid shell truncation of long JWT tokens:

```python
import urllib.request, urllib.parse, json, uuid

# Get token
data = urllib.parse.urlencode({
    "grant_type": "client_credentials",
    "client_id": client_id,
    "client_secret": client_secret
}).encode()
req = urllib.request.Request(f"{instance_url}/services/oauth2/token", data=data)
with urllib.request.urlopen(req) as r:
    token = json.loads(r.read())["access_token"]

# Create session
headers = {"Authorization": f"Bearer {token}", "Content-Type": "application/json"}
body = json.dumps({
    "externalSessionKey": str(uuid.uuid4()),
    "instanceConfig": {"endpoint": instance_url},
    "tz": "America/Los_Angeles",
    "variables": [{"name": "$Context.EndUserLanguage", "type": "Text", "value": "en_US"}],
    "featureSupport": "Streaming",
    "streamingCapabilities": {"chunkTypes": ["Text"]},
    "bypassUser": True
}).encode()
req = urllib.request.Request(f"{api_host}/einstein/ai-agent/v1/agents/{agent_id}/sessions",
                              data=body, headers=headers)
with urllib.request.urlopen(req) as r:
    session_id = json.loads(r.read())["sessionId"]

# Send a message — note: the `message` field in the response is a plain string, not a dict
def send_message(token, session_id, text, seq):
    headers = {"Authorization": f"Bearer {token}", "Content-Type": "application/json"}
    body = json.dumps({
        "message": {"sequenceId": seq, "type": "Text", "text": text}
    }).encode()
    req = urllib.request.Request(
        f"{api_host}/einstein/ai-agent/v1/sessions/{session_id}/messages",
        data=body, headers=headers
    )
    with urllib.request.urlopen(req) as r:
        resp = json.loads(r.read())
    for m in resp.get("messages", []):
        if isinstance(m, dict):
            msg = m.get("message", "")
            if isinstance(msg, str) and msg:
                return msg
            elif isinstance(msg, dict) and msg.get("text"):
                return msg["text"]
    return ""
```

**Always construct message URLs manually** — never use `_links.messages.href` from the session response; it may point to the wrong host for Gov Cloud orgs.

### Step 7 — Run questions in batches

- Collect all non-empty rows from the question column (skip header rows and blank cells) — do NOT filter by any environment label column unless the user explicitly asks
- Use **batches of 25 questions per session** to avoid session context buildup
- Refresh the OAuth token every 30 questions
- Wait 0.5s between messages to avoid rate limiting
- **Save the workbook after every question** — not per batch — so partial progress is never lost if the run is interrupted
- On empty response, retry once with a fresh session before writing `(no response)`

```python
for seq, (row, question) in enumerate(batch, 1):
    response = send_message(token, api_url, session_id, question, seq)
    ws.cell(row=row, column=response_col, value=response)
```

### Step 8 — Semantic comparison (Path B only, if expected answers provided)

After all responses are captured, compare each expected answer against the agent's actual response using semantic judgment — not exact string matching. Paraphrased or equivalent answers should pass.

For each row, write one of three values to the `Pass/Fail (<date>)` column:
- **Pass** — the agent's response clearly conveys the same meaning as the expected answer
- **Fail** — the agent's response is clearly incorrect, missing, or contradicts the expected answer
- **Review** — the response is ambiguous, partially correct, or Claude cannot confidently judge it

### Step 9 — Automatic second pass

Immediately after the first pass completes, run a second pass **without asking the user**. Collect every row where the response is:
- Blank or empty, OR
- Matches a sorry/deflect pattern: starts with or contains `"I'm sorry"`, `"I'm unable"`, `"I cannot"`, `"I apologize"`, or is `"(no response)"`

Re-ask each of those questions in a **fresh individual session** (one new session per question — do not reuse a session across retries, to avoid context carryover). Write whatever the agent returns — even if it's another sorry — as the final answer. Save after each retry.

After the second pass, report:
- How many rows were retried
- How many changed to a substantive response
- How many remained sorry/blank (flag these for human review)

### Step 10 — Save and report

Save the workbook. Report:
- Total questions asked
- Total responses captured
- Blanks (`(no response)`) to review manually
- (Path B with comparison) Pass / Fail / Review counts

Responses are intentionally ungraded in Path A — review them against your agent's grounding data in Salesforce.

### Step 11 — Display results

After saving, ask the user: **"Would you like to view the results here?"**

If yes, or if the user asks for conversation details at any point, display the full conversation log as a single table in this format:

| Conversation | Question # | Question | Agent Response |
|---|---|---|---|
| 1 | 1 | &lt;original question&gt; | &lt;agent response&gt; |
| 1 | 2 | &lt;follow-up text&gt; | &lt;agent response&gt; |
| 2 | 1 | &lt;original question&gt; | &lt;agent response&gt; |
| ... | | | |

Rules:
- **Conversation** — the test question number (1, 2, 3...)
- **Question #** — the turn number within that conversation (1 = original question, 2+ = follow-ups)
- **Question** — the exact text sent (original question or follow-up)
- **Agent Response** — the agent's exact response for that turn
- Never truncate or summarize agent responses
- If a conversation had no follow-ups, it appears as a single row

---

## Key technical notes

- Message type must be `"Text"` (not `"Human"`)
- **Gov Cloud orgs use `https://api.gov.salesforce.com`** — `api_instance_url` in the OAuth response returns `api.salesforce.com` even for Gov Cloud; do not trust it
- **Never use `_links.messages.href`** from the session response — always construct message URLs manually with the correct API host
- Use the **Agentforce Connected App** (e.g. "Headless Agent API"), not the Data Cloud Connected App
- Session key must be unique per session — use `uuid.uuid4()`
- A `410 Gone` on messages means the session expired — create a new one
- Always write the agent's raw response — never summarize or edit it
- Use `openpyxl` for `.xlsx` files; use `csv` module for `.csv` files
- Use Python `urllib` for all API calls to avoid shell truncation of long JWT tokens
- When ending a session (DELETE), always include `x-session-end-reason: UserRequest` header — omitting it returns a 400 error

---

## Error handling

| Error | Fix |
|---|---|
| `410 Gone` on send message | Session expired — create a new session and retry |
| `401 Unauthorized` | Token expired — re-run OAuth and get fresh token |
| `404` on session create | Wrong API host — check if org is Gov Cloud (`api.gov.salesforce.com`) |
| `404` on message send | Do not use `_links` href — construct URL manually with correct host |
| Empty response string | Retry with a fresh single-question session |
| `chatbot_api` missing from scope | Wrong Connected App — use Agentforce app, not Data Cloud app |
| `400` on session DELETE | Add `x-session-end-reason: UserRequest` header to the DELETE request |
| `type: Human not found` | Use `"type": "Text"` |
| Agent not found in BotDefinition | Try partial name match or list all: `SELECT Id, MasterLabel FROM BotDefinition` |
