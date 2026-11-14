# 05 · What the v1.0.3 server source settles

Researched 2026-11-14, offline, from two local checkouts of `CopilotKit/OpenBot`. Nothing was run, installed or fetched.

- **v1.0.3** (commit `8e44272`, 2026-11-13) holds `server/`, `supervisor/`, `agent-computer/`, `worker/`, `toolkit/` and `charts/` under the root MIT `LICENSE`. Every `path:line` below is relative to this checkout.
- **main** (commit `1727413`, 2026-11-14) has none of those directories. A source there is written `main:path:line`.

OpenBot's `channel` is our Conversation, its `agent` is our Bot, its audit trail is our Activity. "Server image" means the published `openbot-server` image.

## The questions

1. **Skins.** How is the saved skin validated and stored, and does a new id survive an unmodified server image?
2. **Serving another frontend build.** Can the server image serve a different build, and what does a separate origin need?
3. **Skills.** Limits, when instruction text reaches the model, whether a Routine can apply one, and what the learned-skill channel carries.
4. **Approvals and policy.** How computer calls are decided, whether one command can be made to ask, which kinds ask, how custom MCP tools are classified, whether "Always allow here" can be switched off.
5. **The Bot's computer.** Contents, workspace, sharing between Bots, network rules, time limits.
6. **Credentials.** How a secret reaches a page, a command and an app tool; custom MCP sign-in; GitHub routes.
7. **Data model.** Tenant columns, the tables, anywhere a Site could attach, and the smallest server change that adds one.
8. **CopilotKit Intelligence.** What it is used for and what the code enforces.
9. **Routines.** Creation, limits, and what happens to an Approval raised by an unattended run.
10. **What moved since.** What `main` says about the server source, and where v1.0.3 is already out of date.

## Short answers

1. **Skins.** The server image carries its own copy of the skin list and answers any other id with a 400. Nothing is stored, and the app then redraws whatever skin the account holds. A fifth id added in a fork's `shared/` does not survive an unmodified server image; an id the server already knows does.
2. **Serving.** Yes. The server serves whatever directory `APP_DIST_DIR` names, and the Helm chart has hooks for extra environment and volumes. A different origin works only as a proxy in front of `/api` and its WebSockets, because the server sends no CORS headers there, and that origin must be listed in `TRUSTED_ORIGINS` or `OPENBOT_APP_URL`.
3. **Skills.** The server sets no limit on instruction length or skill count. A written skill's text reaches the model only when the person's latest message starts with `/slug`: not by matching, not because a Bot carries it, and never in a Routine's run. Learned skills are different: their catalogue is in every run, Routines included, and they carry `SKILL.md` plus files, but the server has no way to publish a hand-written folder into that channel.
4. **Approvals.** Every computer call passes the boundary (refuse or permit only), then the approval gate, then the audit row, then runs. An approval rule cannot single out a command, because every command's scope is the literal `workspace`; only the auto-review model can make one command ask. Seven kinds ask by default, and shell commands and file writes on the Bot's computer are exempt from them. Custom-server tools are always writes. An admin can make a kind ask every time, or add a team rule that asks, and both beat an Operator's "Always allow here".
5. **Computer.** At v1.0.3: Ubuntu 24.04, Node 24.18.1, Bun 1.4.3, Chromium, curl and unzip. No git and no Python. With the Docker supervisor or the Kubernetes sandbox each person has one computer; Helm's default is one computer shared by everybody. Each Bot has its own folder under `/workspace/computers/` on a persistent volume; file tools are confined to it, the shell is not. The network is open by default except cloud metadata addresses. A command gets 120 seconds, 600 at most; an unattended turn gets 5 minutes and 30 tool rounds.
6. **Credentials.** A saved login is typed into the page by the computer after the person confirms that sign-in. Nothing goes from the vault to the shell. An app tool's key is decrypted inside the server and sent in the HTTP header, so it never reaches the Bot's computer. A custom MCP server at v1.0.3 takes one stored key (Bearer, or `Basic …` written out) or Microsoft Entra; per-person OAuth for by-URL servers exists only after v1.0.3. GitHub has no catalogue entry: the routes are Composio, Arcade, or MCP by URL.
7. **Data model.** No organization or tenant column on any Bot, Conversation, Routine, skill, approval or audit table. Nothing free-form is writable through the API. Two unused JSON columns exist on Bots and Conversations, but nothing reads or returns them. The smallest server change is one new schema file (a `sites` table and two link tables), one generated migration and one route module.
8. **Intelligence.** It owns threads, learned skills and recordings, and memory when licensed; the server refuses to start without its address and key. The server's own entitlement code logs warnings, chooses where memory is kept and shows or hides a promotional banner. It enforces no licence, plan, seat or usage limit itself.
9. **Routines.** Made by `POST /api/routines` with a browser session, or by a Bot's `create_routine`; never seeded from the tenant package. Limits: every 15 minutes at most, 20 switched on per person, 2,000 characters of instruction, 5 minutes per firing. An Approval raised by an unattended run never expires: the run ends as `waiting`, the Routine keeps its schedule, and the server carries the action out and continues the turn once the Operator answers.
10. **What moved.** `main` says only that the server, worker, computer and supervisor "run from the published OpenBot images". No reason and no licence statement for those images was found. Already different on `main`: a skin id, CopilotKit 1.88, two migrations, OAuth for MCP by URL, Python on the computer, task threads, and a merged Routines page.

What most changes the plan:

- **A server fork is now possible, and it freezes the server at v1.0.3.** `main`'s app is already built against a newer server (evidence 10), and no later server source is in the repository. A Site table or a new skin id therefore means owning the server and the v1.0.3 app together. The plan that needs no fork still stands: reuse a skin id the server knows, and keep Sites in a store of ours.
- **"Every Ship step asks" is reachable by configuration, but only through app tools.** A team rule of the shape `<app>/*`, `write`, `ask` beats every personal allow, and a kind can be closed to "ask every time". No rule can make one shell command ask, so `git push` stays out of the shell.
- **A Routine is a small box.** No written skill, 2,000 characters, 5 minutes, 30 tool rounds, one Approval at a time. The daily run has to name a file to follow and do one bounded piece of work per firing.

## Evidence

### 1. Skins

- The list is one array, `openbot`, `openduet`, `opendots`, `openmuse`, kept in `shared/` because "the server has to know which values it may store". `shared/user-preferences.ts:10-18`
- The server imports that array and validates `PATCH /api/settings/preferences` with a strict schema: `skin: z.enum(SKIN_IDS)`, unknown keys refused. A body that fails gets a 400 naming the allowed skins, and nothing is written. `server/src/user-preferences-routes.ts:1,12-20,44-54`
- Storage is the `users.preferences` JSON column, merged in Postgres. `server/src/db/schema/core.ts:70-73`, `server/src/user-preferences.ts:59-71`
- On read, a stored skin this build does not know is left in the row but answered as the default. `server/src/user-preferences.ts:10-19`
- The app applies the account's skin over the browser's cached one whenever preferences load, and reads any unknown value as the default. `app/src/skins/account-sync.tsx:19-26`, `app/src/skins/store.ts:19-21`
- The image holds `server/` and `shared/` as source files and starts them with `bun src/production-entry.ts`, so the list an image enforces is the file it was built with. `server/Dockerfile:69-70,116`
- The id has already moved once: `main` ships `openfuse` and maps the old `openmuse` to it. A v1.0.3 server refuses `openfuse`. `main:shared/user-preferences.ts:17,33-35`
- Consequence, by inference: without a server change, our Frame must sit on an id the server accepts, or the fork must stop saving the skin to the account.

### 2. Serving a different frontend build

- `APP_DIST_DIR` is read from the environment at start-up; the image sets it to `/app/app/dist`. `server/src/config.ts:1710-1712`, `server/Dockerfile:84-87`
- When set, the server serves static files from that directory and answers every other non-`/api` path with its `index.html`, read per request. `server/src/app.ts:2319-2340`
- The chart passes extra environment, volumes and mounts to the server pod. `charts/openbot/values.yaml:351,365-366`, `charts/openbot/templates/server/deployment.yaml:165-188`
- There is no CORS on `/api`: "The app's API calls get no CORS from this server, deliberately, so the app has to reach `/api` on its own origin." Only the CopilotKit runtime answers CORS, and only for trusted origins. `server/src/app.ts:2306-2310`, `server/src/copilot.ts:3249-3258`, `server/Dockerfile:3-5`
- The supported shape for two addresses is a proxy: "The app's servers proxy API calls to this one, so the browser's `Origin` names the app's port and the request's `Host` this server's". `server/src/config.ts:698-701`
- Every unsafe `/api` request and every WebSocket upgrade must come from a trusted origin or from the request's own host. `server/src/request-trust.ts:115-128,154-175`, `server/src/app.ts:575`, `server/src/http-server.ts:259-269`
- Trusted origins are the public URL, the app URL, the auth base URL and `TRUSTED_ORIGINS`; Better Auth is given `TRUSTED_ORIGINS` as well. `server/src/config.ts:703-719,1077-1085`, `server/src/auth/index.ts:380-383`
- `OPENBOT_APP_URL` is where sign-in and consent callbacks send the person back. `server/src/config.ts:357-369`
- No cookie-domain or SameSite setting was found in `server/src/auth`, so the session cookie belongs to whichever host the browser sees.
- Side finding: the package's `theme.css` is checked against a list of approved variables at start-up, but no route that serves it was found. `server/src/tenant-package.ts:27-62`

### 3. Skills

- `POST /api/plugins/skills` requires a slug of 2 to 40 characters and a non-empty title and instructions. It checks no length on title, summary or instructions, and no count. `server/src/plugins/route-groups/skills.ts:40-63,117-126`
- `instructions` is an unbounded text column; slugs are unique across the deployment. `server/src/db/schema/plugins.ts:904,912`
- The tenant package parser asks only for non-empty strings. `server/src/tenant-package.ts:317-322,696-731`
- Instruction text enters the prompt in one place, the `picked_skill` section, and only when the run's command names a skill this Bot holds. `server/src/copilot.ts:848-852,1395-1397`, `server/src/plugins/picked-skill.ts:58-71`
- The command is either a `/slug` at the very start of the latest user message, or a slug a server-side host states for its turn. `server/src/plugins/picked-skill.ts:37-52`
- Only the group-conversation host states one, from a person's own message. `server/src/channels/group.ts:835-840,883`, `toolkit/src/run-turn.ts:112-118,405`
- Automatic matching narrows tools, not instructions: a skill's instructions are "Not read by narrowing". It starts above 12 granted tools. `server/src/plugins/selection.ts:36-48,108`
- A Routine's turn is given the instruction and no skill, and the instruction is wrapped in three fixed sentences, so it never starts with `/`. `server/src/routines/runner.ts:160-171`, `toolkit/src/run-turn.ts:843`, `shared/routine-firing.ts:9-13,52-54`
- Learned skills reach every run: "a chat turn, a routine, a channel, a hop". Their catalogue is its own prompt section, and two tools load a skill and read its files. `server/src/learning/runtime.ts:29-38,41-59`, `server/src/copilot.ts:832`
- A learned skill is a name, a description and files with `SKILL.md` first. `shared/learning.ts:251-256`, `server/src/learning/runtime.ts:227-235`
- The server uses seven Intelligence Learning operations: read a container, list and start runs, list and read proposals, read evidence, approve or reject. None creates or uploads a skill. `server/src/learning/review.ts:12-21`
- Its own write routes are approve or reject a proposal, start a run, and open or close a skill to other people. `server/src/learning/routes.ts:800-805,900`, `server/src/learning/sharing-routes.ts:786-800`
- Proposals come from Intelligence's analysis of conversations and recordings. `docs/automatic-learning.md:29`

### 4. Approvals and policy

- One function governs every computer call: "Decide, ask, record, then act." It evaluates the boundary, refuses if denied, asks the approval gate, writes the audit row, then runs. `server/src/computer/gateway.ts:683-691,786,800-803,854-875`
- The boundary is CEL with two lists. A deny match refuses, an allow match permits, nothing else exists. The shipped default allows everything. `server/src/computer/policy.ts:22-41,368-420`, `server/src/computer/policy-store.ts:52-55`
- How each call is classed for the gate. `server/src/computer/gateway.ts:1757-1819`
  - Navigate, read, snapshot, scroll, list and read files: effect `read`.
  - Click, type, key, write file, run command: effect `write`.
  - Scope: a file's folder, the literal `workspace` for any command, otherwise the page's host.
- A rule has five fields and no other: `botId`, `toolRef`, `effect`, `scope`, `behaviour`. Each pattern is a glob where `*` matches any run of characters. `server/src/approvals/routes.ts:14-21`, `server/src/approvals/policy.ts:35-42,80-101`
- So a rule on `computer_run_command` matches every command or none. The command text is in the action's arguments, which rules do not read.
- The auto-review model does read the arguments. Its verdict `needs_approval` makes the action ask, and an admin's guidance "can tell you to ask about more". It fails closed. `server/src/approvals/policy.ts:801-863,865-900,1068-1076`
- Auto-review runs when the person switched it on or the team setting `enforceAutoReview` is on, for every action that is not a read. `server/src/approvals/service.ts:206-212`, `server/src/approvals/policy.ts:728-733`
- The seven kinds: deleting, sending, sharing, publishing, payments, credentials, security settings. `server/src/approvals/types.ts:33-49`
- They are recognised from an app tool's name, a clicked button's label, payment or credential words, or the app's destructive marker. `server/src/approvals/policy.ts:221-275,305-339,596-598`
- Reads never match a kind, and neither do `computer_write_file`, `computer_read_file`, `computer_list_files`, `computer_run_command` or host commands. `server/src/approvals/policy.ts:510-511,527-528`
- A credential typed on a website is always handed to the person. `server/src/approvals/policy.ts:207-208,563-566`
- A task started from a check-in asks before every change, whatever the person's rules say. `server/src/approvals/service.ts:321-338`, `shared/standing-rule.ts:10-12`
- The order of decision: kind, strictest rule, reviewer, then the person's "ask before making changes" switch, which is off by default. `server/src/approvals/policy.ts:946-1139`, `server/src/db/schema/approvals.ts:27`
- An admin can set each kind to `open`, `ask` or `hand_off`. A kind set to `ask` offers "Allow once" and "Deny" only, and the server refuses a standing answer. `server/src/approvals/routes.ts:141-158`, `server/src/approvals/policy.ts:910-933`, `shared/standing-rule.ts:21-30`, `server/src/approvals/store.ts:185-193`
- A team rule that asks beats any personal allow and any "Allow for this conversation". `customRulesEnabled: false` switches personal rules and conversation answers off for the deployment. `server/src/approvals/policy.ts:140-179,975-982`
- Making an app Shared creates exactly this team rule: `toolRef: <app>/*`, `effect: write`, `scope: <app>`, `behaviour: ask`. `server/src/plugins/account-mode.ts:136-142`
- An app tool is judged as `<serverId>/<toolName>` with the server id as scope. `server/src/plugins/plugin-store/calls.ts:611-620`
- A server added by URL has no reviewed list, so "Everything it offers is a write". The server's own `readOnlyHint` is recorded and deliberately not believed. No route or screen that changes a tool's effect was found. `server/src/plugins/catalogue.ts:635-675`, `server/src/plugins/mcp.ts:419-455`
- An allow rule on a named tool is decided before the person's switch, so it stops an ordinary write from asking. It does not reach a kind. `server/src/approvals/policy.ts:64-68,1106-1114,1124`

### 5. The Bot's computer

- Image: `ubuntu:24.04`, with `/usr/local` from `node:24.18.1`, Bun 1.4.3, Playwright 1.62.1 Chromium, and the packages `ca-certificates curl unzip xz-utils`. Nothing installs git or Python. `agent-computer/Dockerfile:6-7,29-32,44-54`
- The v1.0.3 tool text names no installed languages. `main`'s names Python 3 with pip and eight libraries, Node.js, Bun, curl and unzip, and still no git. `app/src/lib/copilot/computer-tools.tsx:787-798`, `main:app/src/lib/copilot/computer-tools.tsx:803-807`
- The computer process runs as root in its own image, so `apt-get` works. The all-in-one image runs it as `pwuser` with `sudo openbot-apt`, which takes package names only. `agent-computer/src/workspace.ts:4-6`, `Dockerfile:52-53,247-256`
- The workspace is `/workspace` on a named volume, with the browser profile on another. `agent-computer/Dockerfile:86-88`, `supervisor/src/docker.ts:540-541`
- An upgrade replaces the container and keeps both volumes, so packages installed with apt are lost and files are kept. A reset removes the profile and keeps the workspace. `supervisor/src/docker.ts:595-600,767-770`
- With the supervisor, one computer per person is the default; an Isolated Bot gets its own, shared by everyone who uses it. `supervisor/src/names.ts:44-49`, `server/src/agents/isolation.ts:4-6`
- Helm's default is `computers.mode: shared`, one computer for every Bot and person, where "a command one Bot runs can read another's files". `sandbox` gives a computer per person. `charts/openbot/values.yaml:370-382`, `server/src/computer/gateway.ts:408-418`
- Each Bot's folder is `/workspace/computers/<id>`, and its shell starts there with `HOME` set to it. `agent-computer/src/workspaces.ts:36-44`, `agent-computer/src/shell.ts:111`
- File tools refuse absolute paths, `..` and symlinks that leave the folder. `agent-computer/src/workspace.ts:9-19`
- The shell is not confined: "A command is still a process in a shared container and can read any path the container can." `agent-computer/src/workspaces.ts:17-20`
- Network: with no policy row the mode is `allow_all`. `server/src/computer/policy-network.ts:43-44`
  - Link-local and cloud metadata addresses are refused in every mode. `agent-computer/src/egress.ts:377-391`
  - `defaults_plus_allowlist` adds 18 hosts, among them `github.com`, `registry.npmjs.org` and `pypi.org`. `agent-computer/src/egress.ts:212-231`
  - The filter is a proxy for the browser and for commands that honour proxy variables; it cannot stop a raw socket. `agent-computer/src/egress.ts:176-181`
  - The browser may not navigate to private or internal hosts unless a local-only switch is on. `server/src/computer/target.ts:279-296`, `docs/configuration.md:859`
- A command: 120 seconds by default, 600 at most, 64 KiB of output returned. `agent-computer/src/shell.ts:26-29,37`
- An unattended turn: 5 minutes of wall clock, set once and not reset by progress. It is a constant with no environment setting. `toolkit/src/limits.ts:8-10`, `toolkit/src/run-turn.ts:1108-1131`
- An unattended turn also stops after 30 rounds of server-run tool calls, and its step ceiling is 48. Chat gets 8 steps per run. `toolkit/src/headless-tools.ts:79,203-206`, `toolkit/src/copilotkit/last-step.ts:64-77`, `server/src/copilot.ts:894-896,915`
- Chat has no duration limit: "A turn may legitimately run for an hour." The only watchdog is for silence, and it is off unless `AGENT_STALL_TIMEOUT_MS` is set. `server/src/channels/turn-watchdog.ts:1-9`, `docs/configuration.md:88`

### 6. Credentials

- **Browser.** The Bot opens a sign-in request and waits. The person types a login, picks a saved one for exactly that site, or takes over the browser. The server passes the login to the computer, which types it. The Bot is told the outcome and never the credential. `server/src/passwords/service.ts:17-38`
- Saved logins are per person and per origin, encrypted, and "never returned by any route". An unanswered request expires after 15 minutes. `server/src/db/schema/passwords.ts:14-49`, `server/src/passwords/service.ts:67`
- **Shell.** A command inherits PATH, locale, terminal and proxy variables, plus any names the operator lists in `COMPUTER_SHELL_ENV`. No code path from the vault to a command was found. `agent-computer/src/shell.ts:50-66,86-111`, `docs/configuration.md:1014-1017`
- **App tool.** The vault row is decrypted in the server and sent as the `Authorization` header, or a catalogue entry's own header, on the call to the MCP server. `server/src/plugins/mcp.ts:383-391`, `server/src/plugins/plugin-store/context.ts:263`, `server/src/credentials.ts:243-250`
- A call whose arguments contain a password or key is refused. `server/src/plugins/plugin-store/calls.ts:454-476`
- So a key held as an `mcp` credential is usable by an app tool and is never on the Bot's computer, provided the computer is a separate container. In the all-in-one image the computer process "holds the deployment's secrets", and only the allow-list above keeps them from a command. `agent-computer/src/shell.ts:54-56`
- **Custom MCP server at v1.0.3.** An admin posts an id, a title, a URL and optionally a `credentialId` or an Entra scope. `server/src/plugins/route-groups/servers.ts:224-239`
- The stored token is sent as `Bearer` unless it already names its scheme: "DataForSEO's hosted server … only speaks Basic", so a token written `Basic <base64>` is sent as written. `server/src/plugins/mcp.ts:359-371`
- Dynamic client registration exists for catalogue apps such as Notion. `server/src/plugins/oauth.ts:462-470`
- Per-person OAuth with registration for a server added by URL is described only on `main`, in a file v1.0.3 does not have. `main:docs/plugins/mcp-by-url.md:6-12,19`
- **GitHub.** The catalogue holds Parallel, Exa, Firecrawl, Google Drive, Notion and Routines, and no GitHub. `server/src/plugins/catalogue.ts:225-490`, `docs/architecture.md:332`
  - Composio: a Shared app is one account for every Bot granted it, "a company GitHub bot". `docs/plugins/composio.md:498,532`
  - Arcade: 43 GitHub actions, and one consent authorises all of them, writes included. `docs/plugins/arcade.md:217,252-253`
  - MCP by URL with a stored key, as above.
  - Linking a GitHub account answers "who is this here?" and is not a tool route. `server/src/db/schema/identity.ts:1-6`, `docs/configuration.md:738-748`

### 7. Data model

- 106 tables in `server/src/db/schema/`, 89 migrations (`0000` to `0088`). Counted with `grep -c "= pgTable("` and from `server/drizzle/meta/_journal.json`.
- No organization, tenant, project, folder or tag column on Bots, Conversations, Routines, skills, approvals or audit rows. A search of the schema for such names finds only `deployment_packages.tenant_id`, `sso_providers.organization_id` and `push_devices.project_id`. `server/src/db/schema/core.ts:238,263`, `server/src/db/schema/delivery.ts:75`
- The tables, by area:
  - Bots: `agents`, `agent_profiles`, `agent_preferences`. `server/src/db/schema/core.ts:271-282`, `server/src/db/schema/coworker.ts:30-91`
  - Conversations: `channels`, `channel_memberships`, `channel_agents`, `attachments`, `group_messages`. `server/src/db/schema/core.ts:284-423,547`, `server/src/db/schema/group.ts:37`
  - Threads: held by Intelligence. OpenBot keeps the thread id per person and Conversation. `server/src/db/schema/core.ts:507-536`
  - Routines: `routines`, `routine_runs`, `routine_sweeps`. `server/src/db/schema/coworker.ts:107-204`
  - Skills: `skills`, `skill_tools`, `plugin_grants`. `server/src/db/schema/plugins.ts:878-1017`
  - Approvals: preferences, team settings, requests, rules, team rules, conversation grants. `server/src/db/schema/approvals.ts:21-208`
  - Activity: `audit_events`, append-only. `server/src/db/schema/core.ts:448-505`
  - Credentials: `credentials`, `saved_logins`, `sign_in_requests`. `server/src/db/schema/core.ts:425-446`, `server/src/db/schema/passwords.ts:22-59`
  - Preferences: `users.preferences`, `user_instructions`. `server/src/db/schema/core.ts:70-73,197-204`
- Where a Site could attach without a schema change:
  - `users.preferences`: no. Unknown keys are refused. `server/src/user-preferences-routes.ts:12-20`
  - A new Conversation takes Bot ids and nothing else; the only text a client can set later is its title. `server/src/channels/routes.ts:1301-1323,1606-1615`
  - `agents.configuration`: built by the server from known fields. An edit keeps keys already there. `server/src/agents/profile-store.ts:699-702,738-741`
  - `agents.override` and `channels.override`: JSON columns that no server code reads or writes. A value put there by SQL would stay, and no API would return it. `server/src/db/schema/core.ts:279,305`
  - `enterprise_settings`: a key and value table, with a fixed set of eight keys in code. `server/src/admin/settings-store.ts:68-77`
  - Component data functions: read-only functions compiled into the server. `server/src/components/functions.ts:36-43`
- The smallest server change that adds a Site:
  1. One new schema file with `sites`, `site_agents` and `site_channels`, exported from the index. The repository's own rule is to add files, "never edit core.ts". `server/src/db/schema/coworker.ts:4-5`, `server/src/db/schema/index.ts:3-27`
  2. One generated migration. Hand edits are forbidden and CI checks for gaps. `docs/development.md:116-150`
  3. One route module mounted beside the others, behind `requireUser`. `server/src/app.ts:863,1925`
  4. Nothing in the run path changes. Bot and Conversation ids are stable text keys. `server/src/db/schema/core.ts:272,287`
- The cost: our own server image, built from one Dockerfile that also builds the app and the worker, and a migration number `main` has already used. `server/Dockerfile:1-10`, `main:CHANGELOG.md:11-14`

### 8. CopilotKit Intelligence

- "Intelligence is a requirement of the product, not a tier: it owns durable threads, memory and learning, and a deployment without it silently forgets every conversation." `server/src/copilot.ts:147-158`
- Start-up check: the API URL, the gateway WebSocket URL and a key must all be set, or the server throws "CopilotKit Intelligence is required and is not configured". `server/src/config.ts:1179-1201`
- `COPILOTKIT_LICENSE_TOKEN` is optional in OpenBot: read and forwarded, never required. `server/src/config.ts:39-50,1174-1184`
- What runs through it:
  - Run events reach the browser over a WebSocket to the Intelligence gateway. `server/src/channels/turn-watchdog.ts:22-26`
  - Unattended turns run on an Intelligence runner and hold its thread lock. `server/src/index.ts:2130-2133`, `toolkit/src/limits.ts:22-41`
  - Learned skills, proposals and recordings. `server/src/learning/review.ts:12-21`
  - Memory, when entitled; otherwise "OpenBot's own tables when Intelligence is unlicensed or unreachable". `server/src/edition.ts:89-91`
- What the entitlement code does:
  - At boot it reads the entitlement, logs a warning if it is not ready and active, and "never blocks start-up". `server/src/intelligence-entitlement-check.ts:18-20,123-195`
  - Its advice says the licence belongs on Intelligence's own deployment: "setting it on OpenBot alone is not enough". `server/src/intelligence-entitlement-check.ts:35-52`
  - Two entitlement fields are read anywhere in the server: `features.memory`, and `planCode` for the self-host banner. `server/src/memory/intelligence.ts:118-121`, `server/src/self-host-banner.ts:15-39`
  - The banner shows unless the plan is `pro`, `team`, `team_self_hosted` or `enterprise`, and `OPENBOT_SELF_HOST_BANNER=false` turns it off. `server/src/config.ts:1541-1544`
- No seat count, usage count, or production-versus-local test on Intelligence was found in the server.

### 9. Routines

- `POST /api/routines` takes `agentId`, `instruction`, `cron`, `channelId` and optional `title` and `timezone`. The owner is the signed-in caller. `server/src/routines/routes.ts:117-150`
- No bearer or API-key scheme was found in `server/src/auth`, so the call needs a signed-in session, or a single-user install, which "admits every request as one administrator with no sign-in". A Bot can also create one with `create_routine`. `server/src/config.ts:644`, `docs/routines.md:25-30`
- "a tenant package cannot yet ship routines". The package's file list has no routines entry. `docs/routines.md:152-155`, `server/src/tenant-package.ts:115-131`
- Limits:
  - 15 minutes between firings. `server/src/routines/schedule.ts:4`
  - 20 switched on per person. `server/src/routines/store.ts:76`
  - 2,000 characters of instruction, 100 of title. `server/src/routines/store.ts:78-80`
  - Ten failures in a row switch it off. `server/src/routines/runner.ts:45`
  - A firing more than 10 minutes late is skipped. `server/src/routines/sweep.ts:58`
  - It runs as its owner, with the owner's grants. `server/src/routines/runner.ts:92-93`
  - It needs the worker process. `docs/routines.md:98-102`
- When an unattended turn reaches an action that asks, the turn stops. The run row is finished as `waiting` and the reason is posted in the Conversation. `server/src/routines/runner.ts:173-182`
- The request row has no expiry column and no expiry code was found in `server/src/approvals/`. `server/src/db/schema/approvals.ts:89-110`
- Once decided, a queue item carries the action out and continues the turn. If the continued turn asks again, it stops again on the new request. When it finishes, the run row becomes `succeeded`. `server/src/approvals/resume-sweep.ts:27-31,129-150,217-246`
- A waiting run is a finished row, so the Routine keeps firing. A later attempt at an equivalent action in the same Conversation is refused while the first is pending. `server/src/approvals/service.ts:344-353`
- A turn that stops to ask a question fails: "a routine has nobody to ask." `toolkit/src/run-turn.ts:677-681`

### 10. What moved since v1.0.3

- The only statements found: the server, worker, computer and supervisor "run from the published OpenBot images", and "there is no chart in the checkout to build from". `main:README.md:174-178`, `main:CHANGELOG.md:220-228`, `main:docs/development.md:88`
- The images are pulled by one tag, `main` on the main branch and a release's version in a release. `main:backend-images.env:1-19`
- No reason for the move was found in `main`'s README, changelog, docs or workflows. No licence statement for the images was found either. `LICENSE` is identical in both checkouts, and `main` still says "The OpenBot application source is MIT-licensed". `main:README.md:107,628`
- Where v1.0.3 is already out of date:
  - Skin id `openmuse` became `openfuse`. `main:shared/user-preferences.ts:17,33-35`
  - CopilotKit moved from 1.85.0 to 1.88.0 on both sides, and the app and server versions move together. `server/package.json:25`, `main:app/package.json:20`, `main:CHANGELOG.md:230-233`
  - Two migrations, `0089` and `0090`. `main:CHANGELOG.md:11-14`
  - MCP by URL gained a probe and per-person OAuth. `main:docs/plugins/mcp-by-url.md:6-12`
  - The computer's tool text lists Python and libraries. `main:app/src/lib/copilot/computer-tools.tsx:803-807`
  - Task threads: a Bot holding `delegate` hands work to a thread of its own. A Routine is offered no such tool. `main:shared/orchestration.ts:11-16,26-27`
  - Asking for a secret no longer clicks the field first. `main:shared/computer-control.ts:91-104`
  - Routines, goals and check-ins are one list, and `POST /api/routines` takes a `goalId`. `main:docs/routines.md:8-14,93-95`
  - Settings and Admin are regrouped and pages renamed. `main:CHANGELOG.md:32`
  - `agent-bot` and `agent-langgraph` are gone from `main`.
- `shared/` files that differ: `audit-words`, `bot-prompt`, `computer-control`, `model-providers`, `research`, `responsibility-turn`, `tool-name`, `user-content`, `user-preferences`. New on `main`: `harness-own-tools`, `orchestration`.
- `docs/` files that differ: `architecture`, `automatic-learning`, `configuration`, `coworkers`, `deployment`, `routines` and the four plugin pages. `approvals.md` is identical. New on `main`: `how-openbot-works.md`, `plugins/mcp-by-url.md`.

## Still unverified

- **The images against this source.** No image was pulled. Whether the published v1.0.3 images match this tree, and how the `main` server behaves on any point above, cannot be read.
- **Terms for the images and for a server fork.** The code says nothing about them. See research 03.
- **Hand-written learned skills.** OpenBot's server cannot publish one. Whether Intelligence's own dashboard or API can is outside this repository.
- **Auto-review as a gate for one command.** The path exists in code and the verdict is a model's. Whether guidance such as "ask before any `git push`" holds needs a test on a running Instance.
- **Unattended limits in practice.** How the 5 minutes, the 30 tool rounds and the 48 steps interact on a real SEO run was read, not measured.
- **The `main` computer image.** Python is in the tool text; the image itself was not inspected, and git is named in neither version.
- **GitHub tools.** Which pull-request and commit tools Composio, Arcade or GitHub's own MCP server offer, and whether GitHub's server accepts a single stored key, are not in the repository.
- **A proxy on another site.** Cookie behaviour there rests on Better Auth's defaults and was not tested.
- **The all-in-one image.** How the operating system keeps a Bot's shell from the server's environment there was not traced beyond the two user accounts.
- **The unused `override` columns.** Nothing says upstream will keep them unused.
- **Sandbox and shared computers on Kubernetes.** Volume and replacement behaviour was read for the Docker supervisor only.
