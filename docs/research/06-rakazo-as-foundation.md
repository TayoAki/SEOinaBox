# 06 · Rakazo as the foundation: how it measures against the same needs

Researched 2026-11-14. Read-only: nothing was run, installed or fetched. This is fact-finding, not legal advice.

How sources are cited:

- `path:line` is elie222/rakazo `main` at commit `964748d` (2026-11-14, pull request #4722), from a shallow clone with one commit.
- `seo-skills:` paths are the Illyism/seo-skills clone.
- A docs page is weaker evidence than code. Citations to `docs/`, `README.md`, `PRODUCT.md`, `VISION.md` and `SECURITY.md` are the project's own description; the rest is code, schema or configuration.
- "Not found" means I searched the clone and found nothing.

## The question

Could Rakazo replace OpenBot as the foundation for SEO in a Box: a white-labelled product we sell, one Instance per client, five Bots directed from a Workbench, with every Ship step waiting for the Operator's Approval and recorded in Activity, and the 30 SEO skills used as they are?

## Short answer

**Verdict: usable as the foundation, as a fork, with one piece of real engineering before it meets the Approval rule.** Rakazo clears the three things that blocked OpenBot: the whole product (server, worker, computers, web app, billing, audit) is Apache 2.0 source in one repository; it runs on Postgres and its own containers with no licence key or vendor service; and it has a tenancy model with row-level security in the database. It shares OpenBot's central gap: a Bot's shell, file writes and browser are recorded but do not ask, and every approval setting belongs to the person using it, so an Operator can switch any of it off. Because the gate is ordinary source we may change, that gap is ours to close. The other risk is the project itself: three months old, labelled beta, one named maintainer, and changing fast.

| Need | Rating | Why |
| --- | --- | --- |
| Licence to sell | has it | Apache 2.0 on every manifest, no CLA, no carve-out; the name "Rakazo" is not licensed, so we must rename |
| No outside dependency | has it | Postgres plus its own images; a model key or a model server is the only outside need; analytics and error reporting are off by default |
| Deployment | has it | Installer and Compose for one VM, production Compose with HTTPS, a Railway template; no Helm chart for the app |
| One Instance per client | has it | Each install is a separate stack with its own secrets; nothing manages many installs |
| Bot computer | has it | Browser, shell, files and desktop; live view and take-over; persistent home |
| Approval before every Ship step | partly | Connected-app tools, email and purchases ask; shell, file writes and browser do not; the default mode lets a model approve; no rule can be locked for the deployment |
| Activity | partly | Tool calls, shell commands, approvals and refusals are stored and exportable; no screen shows the audit log, and it holds no arguments |
| SEO skills as they are | partly | `SKILL.md` with frontmatter is native, but skills are text in the database; scripts are never run; the GitHub import does not match the repository's layout |
| Routines | has it | Schedules, change watches, webhook, Git event and email triggers, run history, an API |
| Several Bots | has it | Group chats, Bot-to-Bot messages, subagents, Bot templates in code |
| Connected apps and credential safety | has it | MCP by URL, OpenAPI, optional Composio or Pipedream; vault secrets are injected server-side and not readable from the shell |
| Sites and Deliverables | partly | A Space is close to a Site; Pages and Artifacts are close to Deliverables; neither carries site details or lists pull requests |
| Workbench layout | partly | Sidebar, chat with cards, right panel with Computer, Routines and Pages exist; the rest is a fork of the web shell |
| Project health | partly | Very active and heavily tested, but beta, three months old, apparently one maintainer |

## Evidence

### 1. Licence and the right to sell

- Root licence is the unmodified Apache License 2.0, "Copyright 2026 Rakazo contributors". It grants the right to "sublicense, and distribute" the work and derivatives. `LICENSE:1-3`, `LICENSE:64-69`, `LICENSE:166`
- Every manifest says `Apache-2.0`: root, all six apps, all fourteen packages, the sandbox supervisor and the updater. `package.json:6`, `apps/api/package.json:4`, `apps/web/package.json:4`, `apps/worker/package.json:4`, `packages/adapters/package.json:4`, `infra/sandboxes/supervisor/package.json:4`, `infra/updater/package.json:5`
- No other licence file, no enterprise or source-available directory, no licence-key check: a search for `LICENSE*`, `COPYING*`, `EULA*`, `ee/`, `enterprise`, `licenseKey` finds only `LICENSE` and `NOTICE`. The hosted-service code (Stripe billing, model credits, marketing site) is in the same tree under the same licence. `docs/self-host-billing.md:7-11`, `apps/www/package.json:5`
- `NOTICE` lists one piece of adapted third-party code (the Grok Bot importer, Apache 2.0 and MIT). Redistribution must keep `LICENSE`, `NOTICE` and mark changed files. `NOTICE:3-11`, `LICENSE:86-112`
- No CLA, DCO or trademark policy file. Contributions fall under the licence's own clause 5. `CONTRIBUTING.md:124-129`, `LICENSE:122-127`
- The licence grants no right to the licensor's names or marks. A sold product needs its own name and logo. `LICENSE:129-132`
- No open-core boundary found. SSO, SCIM, audit log with SIEM forwarding, content encryption and billing are all configured by environment variable in the public Compose file. `infra/compose/docker-compose.images.yml:479-492`, `infra/compose/docker-compose.images.yml:76-79`. The project states it as a rule: "No hosted vendor is required for models, computers, memory, voice, integrations, or another core workflow." `VISION.md:17-21`
- Built-in web search needs no account: it uses keyless DuckDuckGo HTML search unless another provider is chosen. `docs/self-host-integrations.md:8-9`
- Nothing phones home by default: analytics is `none`, error reporting is off without a DSN, and bug reports a Bot files stay in the local database. `.env.example:599`, `.env.example:522-523`, `docs/self-host.md:158-160`
- What still points at the upstream project is branding and defaults, not a dependency: image names default to `ghcr.io/elie222/rakazo/*` and can be overridden; a fork's CI publishes to its own namespace; a built-in "Rakazo" skill links to rakazo.com docs; welcome emails and the OpenRouter referer carry rakazo.com. `infra/compose/docker-compose.images.yml:367`, `docs/self-host-upgrading.md:137-153`, `packages/adapters/src/builtin-skills.ts:62-66`, `packages/auth/src/welcome-email.tsx:10-12`, `packages/adapters/src/openrouter-usage.ts:22`
- One provider rule matters for a sold product: the Claude Code CLI sign-in is refused on any server with more than one account or with billing on, and the docs say Anthropic's rules prohibit third-party apps offering Claude account login. Use API keys. `packages/adapters/src/cli-models/personal-access.ts:8-11`, `docs/model-subscriptions.md:18-21`
- **Plainly:** nothing in the repository restricts selling it, hosting it for other organisations, or rebranding it. Rebranding is in effect required.

### 2. What must run, and where

- Services: Postgres, a one-shot migration container, the API, a worker, the web app, and a computer provider. The worker's queue is Graphile Worker inside Postgres; there is no Redis. `docs/self-host.md:5`, `infra/compose/docker-compose.images.yml:307-309`, `:425-428`, `:445-451`, `:546-553`, `docs/self-host-billing.md:113-114`
- Storage: a `DATA_DIR` volume shared by API and worker; S3-compatible object storage is optional. `docs/self-host-production.md:226-229`, `docs/self-host-storage.md:67-71`
- Required secrets, all generated by the installer: `POSTGRES_PASSWORD`, `POSTGRES_APP_PASSWORD`, `BETTER_AUTH_SECRET`, `ENCRYPTION_KEY`, `SCREEN_PROXY_SECRET`, `SANDBOX_SUPERVISOR_TOKEN`. `docs/self-host-secrets.md:13-20`
- A model: an OpenRouter key, another provider's key, or a model server we run. `docs/self-host.md:13-14`
- **Compose from published images:** one installer script; local Docker computers; the web app binds to loopback, so TLS in front is ours. Needs Docker Engine 26 or newer. `README.md:102-113`, `docs/self-host.md:11`, `docs/self-host.md:49-56`
- **Single VM, production Compose:** Caddy with automatic HTTPS, a host-hardening script, zero-downtime deploys from CI with rollback, encrypted backups. It defaults to E2B for computers so the VM holds no Docker supervisor; an overlay adds local Docker computers. `docs/self-host-production.md:7-19`, `:141-165`, `:172-203`
- **Railway:** a template with one service for API, worker and web, plus Postgres; computers run as Railway sandboxes in a separate environment. The guide says it "is intended for a personal deployment": registration closed, one replica, idle computers stop after ten minutes, three computers per user. It needs a Railway API token. `infra/railway/README.md:5`, `:11`, `:20-26`
- **Kubernetes:** only a computer provider (one pod per computer, with the API and worker normally in the same cluster). No Helm chart or manifests for the app itself: not found. `docs/self-host-sandbox-providers.md:14`, `:133-137`
- Privileges for local Docker computers: the supervisor runs as root and holds the Docker socket, "equivalent to control of the Docker host"; the API does not hold it. Each computer is a non-root container with all capabilities dropped, on its own network, capped at 2 GB, 2 CPUs and 2,048 processes by default. `infra/compose/docker-compose.images.yml:364-371`, `:403`, `docs/self-host.md:88`, `infra/sandboxes/supervisor/src/computer-spec.ts:608`, `:651-652`, `docs/self-host-docker-computers.md:18-22`
- Other providers: `kubernetes`, `incus`, `local-vm` (self-run) and `e2b`, `novita`, `daytona`, `createos`, `boat`, `boxd`, `instabox`, `railway`, `wuying` (remote). `docs/self-host-sandbox-providers.md:11-16`, `:450-460`
- Minimum host size: not stated. The production Compose file is sized "for an 8 GB host"; the images file caps Postgres at 2 GB and the API at 1,536 MB. `docs/self-host-configuration.md:198`, `infra/compose/docker-compose.images.yml:314`, `:458`
- Upgrades: move the image tag and bring the stack up; migrations run before the API starts; a failed command-line upgrade does not roll back by itself. An opt-in updater sidecar can apply and roll back. Tags are `vX.Y.Z`, `latest`, `edge` and `sha-<commit>`. Some upgrades need manual steps (Postgres 18, a coordinated cutover). `docs/self-host-upgrading.md:10-38`, `:65-99`, `:130-133`, `:155-162`

### 3. Tenancy

- Model: users belong to organisations as members; an organisation holds Spaces; a Space holds Bots, chats, memory, computers and connections. `packages/db/prisma/schema.prisma:466`, `:619`, `:635`, `:725`
- Roles: an organisation's creator is its owner, and the audit export is open to "owners and admins"; a Space has an owner and members. In a shared Space, of Bots, chats, memory and routines, "Everyone sees and can change them". A read-only role: not found. `packages/auth/src/index.ts:544-548`, `docs/self-host-audit-log.md:6`, `packages/contracts/src/domain.ts:392`, `docs/spaces.md:47-48`
- No app-level administrator. Accounts in `RAKAZO_ADMIN_EMAILS` may only review plugin submissions, add stdio MCP servers and grant credits. `PRODUCT.md:102-106`, `docs/self-host-accounts.md:18-23`
- Row-level security is real. The app connects as `rakazo_app`, created `NOSUPERUSER NOBYPASSRLS`; policies compare each row's `spaceId` with a per-connection setting. `packages/db/prisma/migrations/20261021140000_row_level_security/migration.sql:15`, `:45-49`, `packages/db/prisma/migrations/20261022120000_row_level_security_tenant_tables/migration.sql:16-45`, `packages/db/src/row-security.ts:218`
- Caveats: with one database login the app warns that it "is not enforced"; the docs give `ALTER ROLE … BYPASSRLS` as an escape hatch; I found no policy on `external_effects`, the table of gated tool calls. `docs/self-host-configuration.md:153-155`, `docs/self-host-upgrading.md:54-57`
- Several clients in one deployment: the design allows it (the tree carries the hosted service's billing and is written for many organisations, and Spaces are "authorization boundaries"). But some things are deployment-wide (the Docker host, shared host mounts, deployment model keys), and one person's models, vault, skills and approval rules follow them into every Space they use. `docs/self-host-billing.md:7-11`, `VISION.md:47`, `docs/self-host-docker-computers.md:170-171`, `docs/spaces.md:29-31`
- One deployment per client is straightforward: each install gets its own random secrets and database. `docs/self-host-secrets.md:13-20`, `infra/railway/README.md:21`
- A fleet of deployments: not found. `docs/fleet-management.md` is about Bots managing other Bots. Per-install helpers exist: the updater sidecar, a diagnostics command that prints no secrets, and account export between installs. `docs/fleet-management.md:1-5`, `docs/self-host.md:147-156`, `docs/self-host-backups.md:317`

### 4. A Bot's computer

- Tools: `browser_navigate`, `browser_snapshot`, `browser_act` on a visible Chromium, `computer_observe` and `computer_act` for the desktop, `shell`, and file tools. `docs/computer-runtime.md:55`, `:61`
- Shared or per Bot: by default each Space has one Team Computer whose files and installed tools all its Bots share; each active Bot gets its own display and Chrome process. A Bot can instead have a Private computer. Team folders "are not security boundaries". `docs/computer-runtime.md:40-42`, `:57`, `packages/db/prisma/schema.prisma:1833`
- Docs disagree on browser sign-ins between Team Bots: the user guide says they are "carried over to the other bots", the runtime doc says logins "are independent". The schema has one browser profile per Bot. `docs/computer.md:31-33`, `docs/computer-runtime.md:42`, `packages/db/prisma/schema.prisma:1797-1809`
- Persistence: the workspace is saved at the end of each run, before a stop and before idle suspension; Docker mounts the home directly. Idle computers stop after 10 minutes by default; 0 disables. `docs/computer-runtime.md:117-119`, `docs/self-host-configuration.md:46-48`
- Watching and taking over: the screen opens beside the chat; "Take control" gives a person an exclusive lease and the Bot's screen actions are refused meanwhile. A person holding control also gets a terminal and a file browser. `docs/computer.md:8`, `docs/computer.md:14-21`, `PRODUCT.md:53-56`, `docs/computer-runtime.md:67-89`
- Limits: a shell command stops after 5 minutes by default; tool calls per turn are unlimited unless set; no run duration limit is stated; an unanswered wait is cancelled after 24 hours; a Bot runs one job on its computer at a time. `docs/self-host-configuration.md:51-52`, `:62-68`, `docs/run-execution.md:25-28`, `:103`

### 5. Approvals

- Every tool call passes one gate, `gateToolCall`. `packages/adapters/src/run-tool-gate.ts:150`
- **Asks by default (mode Ask):** any connected-app, MCP or OpenAPI tool that is declared a write, has a changing verb in its name, or lacks a reading verb; and the built-ins `send_email`, `secret_request` (an HTTP request carrying a saved credential), `service_call`, image and video generation, cloud-agent calls. `packages/core/src/action-approval.ts:70-91`, `:101-105`, `:142-147`, `packages/adapters/src/run-tool-gate.ts:160`, `:283-295`
- **Always a person, in Ask and Auto:** changing another Bot's instructions or model, creating a Space, a routine that watches a command. "Always allow" cannot skip these. `packages/core/src/action-approval.ts:92-99`, `:222-225`, `packages/adapters/src/run-tool-gate.ts:266-268`
- **Purchases ask in every mode**, including Never ask: a pay click the page browser recognises, a saved card, an ask-first login. A webhook run is refused instead. `packages/core/src/action-approval.ts:600-627`, `packages/adapters/src/run-tool-gate.ts:280-282`, `:375-385`
- **Recorded but not gated:** `shell`, `write_file`, `browser_navigate`, `browser_act`, `computer_act`, `open_path`, `launch_app`, `spawn_bot`, `run_subagent` and the schedule tools are on an exempt list. So `git push`, `gh pr create`, a `curl` POST or submitting a web form does not ask. The project lists shell and browser as a known gap for outbound data. `packages/core/src/action-approval.ts:8-36`, `SECURITY.md:41`
- Rules: `always_allow`, `require_approval` or `deny`, matched on a tool, a connector or a site. A block wins; otherwise the most specific rule. A `require_approval` rule on `shell` or `browser_act` makes every such call ask, in Ask and Auto. `packages/contracts/src/rpc.ts:1346-1355`, `packages/core/src/action-approval.ts:449-494`, `:723-745`
- Three modes: Ask, Auto (a model judges each action) and Never ask (only blocks and purchases still apply). `PRODUCT.md:95`, `packages/core/src/action-approval.ts:607-643`, `:697-702`
- **The default is Auto, not Ask.** With no saved preference the mode resolves to Auto, and the reviewer defaults to the deployment's own model; a person is asked only when no reviewer can run or it errors. `packages/adapters/src/run-trust.ts:101-108`, `packages/core/src/action-approval.ts:583-589`, `:634`, `packages/adapters/src/auto-review.ts:118-146`. `PRODUCT.md:98` says "The deployment picks the default approval mode"; I found no setting that names a mode.
- Who can change them: any signed-in user, for their own account or their own copy in a Space. Each change is audited. `apps/api/src/routes/account.ts:293-310`, `:338-356`, `packages/db/src/approval-settings.ts:37-49`, `apps/api/src/security-audit.ts:90-93`
- **A deployment cannot lock a rule.** The deployment settings table has no approval field, and I found no environment variable that sets a mode or a rule (only the reviewer's provider and model); the "Always allow" button and Never ask are open to the Operator. The only things an Operator cannot switch off are the hard-coded ones above. `packages/db/prisma/schema.prisma:782-801`, `.env.example:336-347`
- An approval raised by a scheduled run: a card is posted in the Bot's chat, the run pauses, a notification is sent; a reminder follows after 4 hours and the run is cancelled after 24 hours (both configurable). In a webhook-started run every built-in that changes something is gated, and purchases, and new sites the reviewer did not pass, are refused. `packages/adapters/src/run-tool-gate.ts:790-832`, `packages/core/src/stuck-work.ts:6`, `:11`, `packages/core/src/action-approval.ts:192-194`, `:335-341`

### 6. Activity

- Three stores, all in Postgres:
  - Thread events, including `agent.tool.called`, `agent.tool.completed`, `effect.recorded` and `computer.command` (each shell command, redacted, with the tail of its output). `packages/contracts/src/events.ts:9-46`, `docs/computer-runtime.md:71-75`
  - `external_effects`: one row per changing tool call with its arguments, status, result, and the reviewer's decision, reason and model. Shell calls get a row too. `packages/db/prisma/schema.prisma:1485-1510`, `packages/adapters/src/run-tool-gate.ts:446-461`
  - `audit_events`: per organisation, append-only (Postgres rejects updates). It records `tool.call` with outcome success, failure or denied, `approval.decided`, `approval.auto_reviewed`, `approval.policy.changed` and credential use. It never holds message content or tool arguments. `packages/db/prisma/schema.prisma:827-851`, `packages/adapters/src/run-tools.ts:199-210`, `docs/self-host-audit-log.md:12-15`, `:46-47`, `:52`
- Viewing: a live Activity feed on each Bot's computer; a sidebar Activity list of runs with status and date filters; tool cards in chat, off by default; routine run history. `docs/computer-runtime.md:71`, `docs/bots-and-chats.md:88-89`, `docs/tool-activity.md:8`, `docs/routines.md:75`
- The audit log has no screen in the web app (no reference to it under `apps/web/src`). It is read through `GET /api/audit-events/export` (JSON Lines, owners and admins), a paged `audit.events` call, or a SIEM webhook. `docs/self-host-audit-log.md:63-77`, `packages/contracts/src/rpc.ts:494-496`
- Retention: audit events 365 days by default. Thread events: no retention stated; they go when the chat or Space is deleted. `docs/self-host-audit-log.md:58`, `packages/db/prisma/schema.prisma:1209-1214`

### 7. Skills

- The format is native: a skill is a `SKILL.md` with YAML frontmatter requiring `name` and `description`; unknown keys are kept. `packages/core/src/agent-skill.ts:1-5`, `:64-90`
- But a skill is a database row, not a folder on disk: `content` text, plus `files`, a map of text reference files that only plugin skills have. `packages/db/prisma/schema.prisma:1680-1704`
- Limits: content 100,000 characters, name 80, description 2,000. Plugin skills: at most 40 companion files, 400,000 bytes together, 100,000 bytes each, extensions `md`, `mdx`, `txt`, `json`, `yaml`, `csv` only. `packages/adapters/src/skill-tools.ts:21-23`, `packages/adapters/src/plugin-connectors.ts:66`, `:79-80`, `:87`
- Scripts are never run, and a skill folder that contains one is left out whole: "Skills with scripts or other non-text files are not supported". The user guide says instead that such skills install as "instructions only". `packages/adapters/src/plugin-connectors.ts:284-287`, `:315-319`, `PRODUCT.md:65`, `docs/skills.md:26`
- Installing: write one in the app, create it through the `agentSkills.create` call, let a Bot save one, or import a plugin from a public GitHub URL. The import needs a plugin manifest and skills at `skills/<name>/SKILL.md`. `packages/contracts/src/rpc.ts:1077-1096`, `packages/adapters/src/plugin-connectors.ts:72`, `:200-201`, `:303`, `:540`
- Attaching: skills are not attached to a Bot. They belong to the person and are offered to all their Bots, in all their Spaces or chosen ones; the Bot model has no skill list. `docs/skills.md:19`, `PRODUCT.md:72`, `packages/db/prisma/schema.prisma:973-1017`
- Invoking: a catalogue of up to 50 names and descriptions is added to each turn and the Bot calls `skill_read`; a person types `/Name`; a routine prompt names a skill as `@Name` and it is loaded when the routine fires. `packages/core/src/agent-skill.ts:125-136`, `:145-156`, `packages/adapters/src/skill-tools.ts:137-167`
- **The 30 SEO skills, as they are:**
  - The repository has a Cursor plugin manifest, but its skills sit at `seo/<name>/`, `content/<name>/` and so on, which the importer's `skills/<name>/` pattern will not find. `seo-skills:.cursor-plugin/plugin.json`, `packages/adapters/src/plugin-connectors.ts:303`
  - 27 folders hold only `SKILL.md`; `content/intro` adds one Markdown file; `seo/internal-linking` and `seo/domain-migration` each add one script (`.mjs`, `.sh`), which would exclude them from a plugin import. Both print the script's full source inside `SKILL.md`. `seo-skills:seo/internal-linking/SKILL.md:31`, `seo-skills:seo/domain-migration/SKILL.md:152`
  - All 30 `SKILL.md` files are under 18,000 characters with descriptions under 530, and their frontmatter uses shapes the parser handles (one-line or block descriptions, one nested `metadata` map). Read against `packages/core/src/agent-skill.ts:422-490`; not executed.
  - So: each `SKILL.md` can be loaded unchanged as text, through the API or a repackaged plugin. The folders cannot be used as folders, and no script runs unless the Bot writes it out and runs it in its shell.

### 8. Bots, models and working together

- A Bot is a row: name, description, instructions (up to 20,000 characters, with history), model provider, model and thinking level, disabled built-in tools, Team or Private computer, memory scope. `packages/db/prisma/schema.prisma:973-1017`, `packages/contracts/src/domain.ts:444`
- One engine: Pi, running inside the API and worker, not in the computer. No per-Bot choice of harness. `docs/self-host-configuration.md:44`, `docs/computer-runtime.md:13`, `packages/adapters/package.json:37-38`
- Models: API keys for OpenAI, Anthropic, Google, OpenRouter and many others; an OpenAI-compatible server URL for a model we host; deployment keys in the environment; ChatGPT sign-in. A main model with fallbacks, a separate approval model, and per-Bot and per-routine overrides. `docs/models.md:10-14`, `:25-43`, `infra/compose/docker-compose.images.yml:104-106`
- Working together: group chats of 2 to 10 Bots with @mentions; Bots message each other (6 hops at most); subagents run inside a turn and the parent waits for them; a Bot can create and manage other Bots. `docs/groups.md:8-17`, `:22-29`, `:35-39`, `docs/self-host-configuration.md:54`, `PRODUCT.md:42`, `docs/fleet-management.md:30-45`
- Ready-made Bots are a list compiled into the code; each adds instructions, a skill and a paused routine. Five SEO Bots would be five entries. `packages/core/src/bot-templates.ts:9`, `docs/bots-and-chats.md:25-26`
- External agents: MCP servers; a one-time import of an OpenClaw, Hermes or Grok Bot agent's setup; cloud coding agents (Cursor, T3 Code). AG-UI or ACP: not found. `docs/integrations.md:27-31`, `NOTICE:3-7`, `docs/self-host-integrations.md:282-300`

### 9. Routines

- Triggers: a schedule (presets, an interval, or cron, in the owner's timezone); a change watch every 5 minutes to a day on files, a command's output or a connected-app tool; a webhook with a URL and key; a GitHub webhook; email to the Bot's own address; a Slack message. `docs/routines.md:30-63`, `packages/db/prisma/schema.prisma:1523-1536`, `apps/api/src/webhook.ts:32-33`
- Limits in the Bot's scheduling tool: 1 minute minimum interval, 50 active routines per Bot. Whether the app's own create call enforces the same was not checked. `packages/adapters/src/schedule-tools.ts:225`, `:457`
- Each routine can have its own model. Run history, a test run and a pause switch exist; a run that starts late is labelled. `docs/routines.md:25-26`, `:73-76`, `docs/run-execution.md:85-90`
- API: `routines.create`, `update`, `remove`, `testRun`, `history`. Bots create them with `schedule_create`; templates seed them paused. `packages/contracts/src/rpc.ts:935-1008`

### 10. Connected apps and credentials

- Custom sources work on any server: an MCP server at an HTTPS address (with OAuth, a bearer token or a header key), an OpenAPI or GraphQL description, a plugin from GitHub. Stdio MCP servers are off unless the operator allows named commands. `docs/integrations.md:25-34`, `docs/self-host-integrations.md:73-75`
- App catalogues (Gmail, GitHub, Slack and others) need a Composio key or Pipedream credentials; both are optional third-party services. `README.md:169-173`, `docs/self-host-integrations.md:137-140`
- GitHub: through a catalogue or a GitHub MCP server, plus the Git event trigger. WordPress: not found anywhere in the product code or docs; it would be an MCP server, an OpenAPI source, or a vault credential. `docs/integrations.md:3`, `docs/routines.md:46-49`
- Once connected, an app serves every Bot in the Space; MCP servers can be limited to chosen Bots, and a Bot asks before first using one. `docs/integrations.md:15-16`, `:60-64`
- Credentials are encrypted in Postgres (or kept in Infisical) and never returned by the API. `README.md:171-173`, `docs/bot-secrets.md:19`, `.env.example:95`
- **A secret usable by a tool but unreadable from the shell: yes.** A vault credential is bound to one HTTPS origin; the Bot calls `secret_request` and the server injects it as a bearer token, header or basic-auth password. "The model has no tool for reading these values", and they are deliberately not exposed to shell commands, files or environment variables. That call asks for approval by default. `docs/bot-secrets.md:21-39`, `packages/core/src/action-approval.ts:75`
- The opposite also exists: a "command variable" is exported into the Bot's shell, and the docs warn "The bot's shell reads this value." Pushing with `git` from the shell would need one. `docs/bot-secrets.md:56`, `:64`
- Each vault item can be limited to named Bots or blocked; each use is audited. `docs/bot-secrets.md:161-163`, `docs/self-host-audit-log.md:17`

### 11. Sites and Deliverables

- A Space is the closest thing to a Site: "a separate workspace with its own bots, chats, memory and computer", suggested for "a client". App connections, the Team Computer and saved skills stay inside it. It has a name and nothing else: no repository or site address. `docs/spaces.md:3-4`, `:11-12`, `:16-22`, `packages/db/prisma/schema.prisma:635-644`
- Spaces appear in the sidebar as headed groups of Bots once there is more than one, not as a switcher. `apps/web/src/pages/shell/space-sidebar.ts:42-48`
- Things a Bot produced live in two places. Pages: reports and dashboards a Bot builds, versioned, listed in a Pages tab per Bot, shareable by link. Artifacts: every file a Bot sent, listed per Space with search and a date filter. `docs/pages.md:15-22`, `packages/db/prisma/schema.prisma:1948-1983`, `:1985-2028`, `docs/bots-and-chats.md:90`, `packages/contracts/src/rpc.ts:1394-1411`
- A pull request or a WordPress change is not a Page or an Artifact; it exists only as an `external_effects` row and a chat message.
- To add: site details on a Space (or a new table) and a switcher; one Deliverables list drawing on Pages, Artifacts and shipped effects.

### 12. The interface

- Stack: React 19, Vite, Tailwind 4, React Router, TanStack Query, oRPC client, Lingui (15 languages), shadcn/ui components vendored into `packages/ui-web`. `apps/web/package.json:19-47`, `README.md:303-305`, `AGENTS.md:29`
- Size: 377 source files and about 67,600 lines under `apps/web/src`, excluding 233 unit-test files and 124 browser specs. The shell is `Shell.tsx` (1,441 lines) plus 76 files (18,301 lines) under `pages/shell/`.
- Stock layout: a resizable left sidebar (search, Activity toggle, create menu, Bots grouped by Space and section, a footer with the account menu) that can shrink to a strip of icons; a main column with a chat header, the conversation and a composer; a right panel whose tabs are Computer, Routines, Pages and Advanced. Approvals, questions and results are cards in the conversation. `apps/web/src/pages/shell/sidebar-header.tsx:44-50`, `:66-84`, `apps/web/src/pages/shell/bot-panel.tsx:536-545`, `apps/web/src/lib/right-panel-state.ts:1-10`, `docs/bots-and-chats.md:51-53`
- Against the Workbench: the sidebar with Bots, the conversation with result and approval cards, the composer, the live computer and routines in a right panel all exist. Missing: a separate icon rail, a Site switcher, the three tabs over the main area, a pending-approvals list, an activity pane beside the computer, a bottom status bar.
- Theming: `RAKAZO_THEME` points at a JSON file that overrides colour tokens, fonts and corner radius, web only. Product name, logo and icons have no setting. `docs/self-host-clients.md:135-161`, `apps/web/src/instance-theme.ts:28-34`
- Extension points: `RAKAZO_NAV_LINKS` adds up to 10 account-menu items that frame a same-origin page; marked trusted, the page acts with the signed-in user's access. No plugin mechanism for the layout. `docs/self-host-clients.md:183-199`, `packages/contracts/src/nav-links.ts:20-32`
- Estimate (mine, from the measured files): the Workbench reworks about 15 files and 6,600 lines (`Shell.tsx`, `shell-sidebar.tsx`, `sidebar-roster.tsx`, `sidebar-header.tsx`, `SidebarFooter.tsx`, `space-sidebar.ts`, `use-space-switch.ts`, `side-panel.tsx`, `bot-panel.tsx`, `computer-panel.tsx`, `chat-header.tsx`, `use-right-panel.ts`, `Artifacts.tsx`, `ActivityList.tsx`, `RoutineList.tsx`), about a tenth of the web source, plus new components and their tests. Renaming touches about 137 non-test files (roughly 710 lines) across web, API, adapters, auth and contracts, 52 lines in the English catalogue, and the icon set. At upstream's merge rate (section 13) a fork of these files should be expected to conflict often; that is an inference, the clone has no history to measure it.
- API: a typed oRPC contract (1,718 lines of Zod schemas) served at `/rpc`, shared by the web, desktop and mobile apps. Sign-in is by session cookie. No OpenAPI document and no API tokens for outside clients: not found. `packages/contracts/src/rpc.ts`, `apps/api/src/app.ts:835`, `README.md:254`, `docs/self-host-upgrading.md:42`

### 13. Project health

- Dates: first changelog entry `0.1.0-beta` on 2026-08-13; latest release `0.11.0` on 2026-11-12; head commit 2026-11-14. `CHANGELOG.md:824`, `CHANGELOG.md:9`
- Cadence: 18 versions in 13 weeks, eight of them in November. The head commit is pull request #4722. `CHANGELOG.md:9-824`
- Tests: 1,678 `*.test.ts(x)` files (about 19,000 cases by a text count) and 129 browser specs; offline unit tests, Postgres integration tests, end-to-end and nightly suites. `CONTRIBUTING.md:27-39`
- Maintainers: one named maintainer; review bots and agent instructions are part of the workflow. `CONTRIBUTING.md:137`, `.github/ISSUE_TEMPLATE/config.yml:9-11`, `.coderabbit.yaml:5-6`
- Stability: security fixes cover "the current `main` branch and the latest release (beta)"; the docs say not to assume a `latest` tag "until a stable release exists"; the root version is still `0.1.0`. No breaking-change policy: not found. The database has 280 migrations, and migrations must stay compatible with the previous release. `SECURITY.md:48`, `docs/self-host.md:46-47`, `package.json:5`, `docs/self-host-production.md:159-160`
- Hosted service: its billing, credits and experiments are in this tree, and the code refers to it ("the hosted service does both"). The docs describe deploying each `main` commit to a public server from CI and say "Product contracts stay compatible across cloud and self-hosted". Whether rakazo.com runs exactly this code is not shown. `docs/self-host-billing.md:7-11`, `packages/adapters/src/cli-models/personal-access.ts:14-15`, `AGENTS.md:9`, `docs/models.md:21-23`, `docs/self-host-production.md:141-147`, `docs/self-host-upgrading.md:42`, `VISION.md:21`

## What we would have to build

1. **A deployment-level approval policy.** Fix the mode to Ask, force named kinds of action to ask, and remove Never ask, Auto and "Always allow" for those kinds from the Operator. This is new code in `action-approval.ts`, `run-tool-gate.ts`, `approval-settings.ts` and the settings screens.
2. **A Ship-step path that cannot bypass the gate.** Either make `shell` and `browser_act` ask (every command, which is heavy) or keep write credentials out of the shell so a pull request or a WordPress change can only go through a gated connector tool or `secret_request`. Egress rules do not help: computers reach any public host.
3. **Sites.** Site details on a Space, a switcher, and the connection to a repository or a WordPress site.
4. **Deliverables.** One list over Pages, Artifacts and shipped effects.
5. **Activity.** A screen over `audit_events` and `external_effects` showing what ran, what was refused and who approved, with arguments.
6. **The Workbench layout**, as a fork of the web shell (section 12), or a separate client on the RPC contract.
7. **Rebranding.** Name, logo, icons, emails, the built-in "Rakazo" skill, docs links, our own image registry.
8. **Skill and Bot seeding.** Load the 30 `SKILL.md` files (through the API or a repackaged plugin), decide what to do about the two scripts, and add five Bot templates with their routines.
9. **Fleet tooling.** Provisioning, upgrading and monitoring many Instances; none exists.
10. **Switching off what we do not sell.** Open sign-up, billing, messaging apps, "use your own computer", cross-Space Bot messages, Bots managing Bots.

## Could not verify

- Anything on GitHub itself: stars, number of contributors, issue and release history, whether release tags and images exist. The clone has one commit and no network was used.
- Licences of the npm dependencies, including the Pi engine (`@earendil-works/pi-*`), and of software in the computer image (Chromium, noVNC, agent-browser). Not audited.
- Whether "Rakazo" is a registered mark and whether a trademark policy exists outside the repository.
- Whether the hosted service runs exactly this code or carries private additions.
- All runtime behaviour. In particular the SEO skills were checked against the parser by reading, not by running it.
- Which doc is right about browser sign-ins shared between Team Bots (section 4).
- A minimum host size; none is stated.
- Terms of the optional outside services (Composio, Pipedream, E2B, Railway sandboxes) for use inside a product we sell.
- Why migration folder names carry dates up to 2026-12-19, later than the head commit.
