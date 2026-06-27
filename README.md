# @chuks/connector

Pre-built, typed, authenticated **connectors** to external services for Chuks. Auth is handled, the API is wrapped, and every connector is **agent-ready**: call `.asTool()` and its operations become tools you hand straight to [`@chuks/agent`](https://github.com/chuks-lang/chuks_agent).

In-process, single-binary, no separate server per service. The whole thing compiles into your Chuks binary.

```chuks
import { connectors } from "pkg/@chuks/connector"

// Auth handled. API wrapped. Types enforced. Zero auth code.
var github = connectors.github(githubToken)
var slack  = connectors.slack(slackBotToken)

var issues = await github.issues("chuks-lang/chuks", { "state": "open" })
await slack.send("#eng", "There are " + string(length(issues)) + " open issues")
```

> Status: early. The core (credentials, transport, OAuth refresh, scoped asTool bridge) is stable and tested. The catalog ships **GitHub**, **Slack**, **Linear**, **Notion**, **Google Calendar**, **Gmail** (OAuth), **Discord**, **Stripe**, **Twilio**, **Jira**, and **Microsoft 365** (Outlook, Teams, OneDrive), and grows from there.

---

## Why connectors

OAuth and API plumbing are why nobody builds integrations: register the app, exchange the code, store the token, refresh before expiry, handle revocation, parse each service's quirks. `@chuks/connector` does that once, behind a typed surface, so you write the operation you actually care about.

| | Zapier | n8n | LangChain tools | MCP servers | **@chuks/connector** |
| --- | --- | --- | --- | --- | --- |
| Programmable | no | partial | yes | yes | **yes** |
| Typed operations | no | no | erased at runtime | varies | **compile-time** |
| Separate process | hosted | yes | no | **yes (per server)** | **no, in-process** |
| Agent-native | no | no | yes | yes | **yes, `.asTool()`** |
| Single binary | no | no | no | no | **yes** |

---

## Install

```bash
chuks add @chuks/connector
```

---

## Connectors

### GitHub

Authenticate with a personal access token (or a GitHub App installation token). `repo` is `"owner/name"`.

```chuks
var gh = connectors.github(token)

var issues = await gh.issues("chuks-lang/chuks", { "state": "open", "limit": 20 })
await gh.createIssue("chuks-lang/chuks", "Bug: crash on empty input", "Steps to reproduce ...")
await gh.comment("chuks-lang/chuks", 123, "Looking into this now")
var prs = await gh.pullRequests("chuks-lang/chuks", { "state": "open" })
```

### Slack

Authenticate with a bot token (`xoxb-...`). Slack returns `{ "ok": false }` on logical errors; each method raises a clean error in that case.

```chuks
var slack = connectors.slack(botToken)

await slack.send("#eng-alerts", "Deploy complete")
var msgs = await slack.history("C0123456", { "limit": 50 })
await slack.react("C0123456", "1700000000.000100", "white_check_mark")
```

### Linear

Authenticate with a personal API key. Operations run over Linear's GraphQL API; logical errors surface as clean throws.

```chuks
var linear = connectors.linear(linearApiKey)

var issues = await linear.issues({ "limit": 25 })
var teams  = await linear.teams()
await linear.createIssue(teamId, "Bug in auth", "Steps to reproduce ...")
```

### Notion

Authenticate with an internal integration token. The required `Notion-Version` header is handled for you.

```chuks
var notion = connectors.notion(notionToken)

var hits = await notion.search("meeting notes", { "limit": 10 })
var page = await notion.page(pageId)
var rows = await notion.database(databaseId)
```

### Google Calendar (OAuth)

Pass an OAuth access token and an optional `refresh` callback (`fn(): string`) that returns a fresh token. When the API answers `401`, the transport calls `refresh()` once and retries, so token rotation is invisible. Run the initial authorization-code flow with [`chuks_oauth`](https://github.com/chuks-lang/chuks_oauth), then hand the token and a refresh closure here.

```chuks
var cal = connectors.googleCalendar(accessToken, function(): string {
    return refreshWithOAuth()   // return a new access token
})

var events = await cal.listEvents("primary", { "maxResults": 10 })
await cal.createEvent("primary", "Standup", "2026-06-10T15:00:00Z", "2026-06-10T15:15:00Z")

// No refresh? Pass null.
var readOnly = connectors.googleCalendar(accessToken, null)
```

### Gmail (OAuth)

Same OAuth shape as Google Calendar. `inbox` enriches each message with From / Subject / snippet; `send` builds the message and base64url-encodes it for you.

```chuks
var gmail = connectors.gmail(accessToken, refreshFn)

var unread = await gmail.inbox({ "unread": true, "maxResults": 10 })
await gmail.send("user@example.com", "Re: your question", "Thanks, on it!")
```

### Discord

Authenticate with a bot token (`Authorization: Bot ...` is handled for you).

```chuks
var discord = connectors.discord(botToken)

await discord.send(channelId, "Deploy finished ✅")
var recent = await discord.messages(channelId, { "limit": 50 })
```

### Stripe

Authenticate with a secret key. Stripe uses form-encoded request bodies (not JSON); the transport handles that, so you just pass fields.

```chuks
var stripe = connectors.stripe(secretKey)

var customers = await stripe.customers({ "limit": 100 })
var invoices  = await stripe.invoices(customerId, { "limit": 10 })
await stripe.createCustomer("ada@example.com", "Ada Lovelace")
```

### Twilio

Authenticate with an Account SID + Auth Token (HTTP Basic). Form-encoded under the hood.

```chuks
var twilio = connectors.twilio(accountSid, authToken)

await twilio.sendSms("+15550001111", "+15552223333", "Build is green ✅")
var sent = await twilio.messages({ "limit": 20 })
```

### Jira

Authenticate with your account email + an API token (HTTP Basic). Pass your site base URL.

```chuks
var jira = connectors.jira("https://your-company.atlassian.net", email, apiToken)

var issues = await jira.searchIssues("project = ENG AND status = 'To Do'", { "limit": 25 })
await jira.createIssue("ENG", "Investigate flaky test", "Bug")
await jira.comment("ENG-123", "On it.")
```

### Microsoft 365 — Outlook, Teams, OneDrive (OAuth)

All three run over the Microsoft Graph API and share the OAuth shape: an access token plus an optional `refresh` callback, refreshed transparently on a `401`.

```chuks
var outlook  = connectors.outlook(accessToken, refreshFn)
var teams    = connectors.teams(accessToken, refreshFn)
var oneDrive = connectors.oneDrive(accessToken, refreshFn)

var unread = await outlook.inbox({ "unread": true, "maxResults": 10 })
await outlook.send("user@example.com", "Re: your question", "Thanks!")

await teams.send(teamId, channelId, "Deploy complete ✅")
var recent = await teams.messages(teamId, channelId, { "limit": 20 })

var files = await oneDrive.files({ "limit": 20 })
await oneDrive.createFolder("Reports")
```

---

## Permissions — read vs write

An agent that can `github.createIssue` or `slack.send` is a loaded weapon. Every tool is tagged `read` or `write`, so you can hand an agent the narrowest set it needs.

```chuks
import { asTools, readOnlyTools, selectTools } from "pkg/@chuks/connector"

asTools([github, slack])                  // everything
readOnlyTools([github, slack])            // only read-scoped tools (observe, never mutate)
selectTools([github], { "exclude": ["github_comment"] })
selectTools([github], { "include": ["github_issues", "github_pull_requests"] })
```

A read-only triage agent that can look but not touch is then one line:

```chuks
var watcher = agent.create({
    "model": ai.openai("gpt-4o"),
    "tools": readOnlyTools([github, linear]),   // cannot create, comment, or send
    "systemPrompt": "Report what needs attention. Do not modify anything."
})
```

---

## Use a connector as an agent tool

`.asTool()` turns a connector's operations into tool configs that `@chuks/agent` accepts directly. No glue code.

```chuks
import { agent } from "pkg/@chuks/agent"
import { ai } from "pkg/@chuks/ai"
import { connectors, asTools } from "pkg/@chuks/connector"

var gh    = connectors.github(githubToken)
var slack = connectors.slack(slackToken)

var triager = agent.create({
    "name":  "triager",
    "model": ai.openai("gpt-4o"),
    "tools": asTools([gh, slack]),   // every operation becomes a tool
    "systemPrompt": "Triage open issues and report a summary to Slack.",
    "maxSteps": 8
})

var result = await triager.run("Summarize open issues in chuks-lang/chuks and post to #eng", null)
println(result.answer)
```

Each operation becomes a named tool with a typed JSON-Schema parameter list (`github_issues`, `github_create_issue`, `slack_send`, ...), so the model gets a precise, authenticated interface to the service.

---

## Authentication

Most connectors take a token or API key directly. The credential layer abstracts the differences:

```chuks
import { bearer, apiKey, oauthToken } from "pkg/@chuks/connector"

bearer("ghp_...")                                   // Authorization: Bearer ...
apiKey("k_live_...", { "header": "X-Api-Key" })     // custom header
apiKey("k_...", { "query": "api_key" })             // sent as a query param
oauthToken(accessToken, { "refresh": refreshFn })   // refreshed transparently on 401
basic("user", "pass")                               // Authorization: Basic ...
```

For OAuth services, pair this with [`chuks_oauth`](https://github.com/chuks-lang/chuks_oauth) for the initial authorization-code flow, then hand the access token (and a `refresh` callback) to the connector. Token rotation is invisible after that: the transport refreshes once on a `401` and retries.

### Pluggable token storage

`MemoryTokenStore` ships by default. The contract is two methods, `get(key)` and `set(key, value)`, so you can back it with `std/db` or `chuks_redis` without changing connector code.

---

## What you get per call

Operations return parsed results on success and throw a normalized error on failure (which `@chuks/agent`'s tool isolation catches cleanly). Under the hood the shared transport handles:

- auth + JSON headers,
- retry on transient failures (`429`, `5xx`),
- OAuth refresh on `401`,
- form-encoded bodies for APIs that need them (Stripe, Twilio),
- error normalization across the common API shapes.

---

## Catalog & roadmap

The goal is broad domain coverage. Shipped connectors are marked ✅; the rest are
the most-requested targets, and they are designed to be easy community
contributions — the shared core already handles bearer / API-key / OAuth-refresh
/ HTTP-Basic auth and JSON / GraphQL / form-encoded bodies, so a new connector is
usually just typed operation methods plus an `asTool()` manifest.

**Productivity & Communication**
- ✅ Gmail, ✅ Google Calendar, ✅ Outlook, ✅ Teams, ✅ OneDrive, ✅ Slack, ✅ Discord, ✅ Notion
- Google Drive · Confluence · Airtable · WordPress.com · Zoom · Google Docs/Sheets

**Project Management**
- ✅ Linear, ✅ Jira
- Asana · Trello · Monday.com · ClickUp · Basecamp · GitHub Projects

**Software Development**
- ✅ GitHub
- GitLab · Bitbucket · Vercel · Netlify · CircleCI · Sentry · PagerDuty

**Payments & Business**
- ✅ Stripe
- Paddle · HubSpot · Salesforce · Pipedrive · QuickBooks · Shopify

**Communications & Messaging**
- ✅ Twilio (SMS)
- SendGrid · Mailgun · Postmark · Telegram · WhatsApp Business · Intercom

**Data & Storage**
- Airtable · Google Sheets · Snowflake · BigQuery · Supabase · Dropbox · AWS S3

**AI & Search**
- OpenAI · Anthropic · Perplexity · Algolia · Pinecone · Tavily

### Contributing a connector

1. Add `src/connectors/<service>.chuks` with a connector class: typed operation
   methods (using the shared `HttpTransport` from `core.chuks`) and an `asTool()`
   returning `[]ToolConfig`, each tool tagged `read` or `write`.
2. Wire it into the `ConnectorsModule` facade and the export block in `index.chuks`.
3. Add its host to `declaredDomains` and a small `asTool` test.

Pick the auth helper (`bearer` / `apiKey` / `oauthToken` / `basic`) and body
style (`postJson` / `postFormData`) that the API needs; the transport handles
retries, OAuth refresh on 401, and error normalization for you.

---

## Package layout

```
src/
  core.chuks               Credential, HttpTransport, config dataTypes, ToolConfig
  connectors/github.chuks  GitHubConnector   (REST)
  connectors/slack.chuks   SlackConnector    (REST)
  connectors/linear.chuks  LinearConnector   (GraphQL)
  connectors/notion.chuks  NotionConnector   (REST + version header)
  connectors/google_calendar.chuks  GoogleCalendarConnector (OAuth)
  connectors/gmail.chuks   GmailConnector    (OAuth, base64url MIME)
  connectors/discord.chuks DiscordConnector  (REST, Bot auth)
  connectors/stripe.chuks  StripeConnector   (REST, form-encoded)
  connectors/twilio.chuks  TwilioConnector   (Basic auth, form-encoded)
  connectors/jira.chuks    JiraConnector     (Basic auth, JSON + ADF)
  connectors/microsoft.chuks  Outlook / Teams / OneDrive (Graph, OAuth)
  index.chuks              the `connectors` facade, tool selection, re-exports
```

Each connector lives in its own file and imports the shared layer from `core.chuks`. To add one: create `connectors/<service>.chuks` with a connector class (typed operations + an `asTool()` returning `[]ToolConfig`), then wire it into the `ConnectorsModule` facade in `index.chuks`.

---

## License

MIT
