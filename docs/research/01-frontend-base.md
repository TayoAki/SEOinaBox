# 01 · Frontend base for the Workbench

Researched 2026-11-14 against a shallow clone of CopilotKit/OpenBot `main` at commit `1727413` (root `package.json` version 1.0.3, plus unreleased changes). Every source is `path:line` relative to the OpenBot repo root. Nothing was run, installed or fetched. OpenBot's own words for things are kept where they name its code: its `channel` is our Conversation, its audit trail is our Activity, its `agent` is our Bot, and its "tenant package" is the folder of YAML an Instance is configured with.

## The question

Which is the better base for the six-region Workbench (icon rail, sidebar with Site switcher and Bots, Chat / Deliverables / Routines tabs, the Conversation, a right panel, a status bar), and what does each cost?

- **A.** Fork OpenBot's `app/` and reshape its shell.
- **B.** Build our own frontend against the OpenBot server, reusing pieces of `app/`.
- **C.** Get the layout from OpenBot's own extension points, with no fork.

## Short answer

**Fork (A), but build the Workbench as a skin `Frame`, not by rewriting the stock shell.** OpenBot already has a seam that replaces the whole frame around every page, and one shipped skin (OpenDots) uses it to draw four of our six regions in about 2,000 lines without touching a page. Our Workbench is a new directory beside it, plus a few lines in upstream files.

The main reason: the Conversation is not a transcript on top of an API. It is roughly 300 files of browser-side behaviour (the Bot's computer tools are registered in the browser, as are the inline cards and the Approval cards) bound to a server API of which 6 operations are specified and about 200 paths are not. A separate frontend (B) has to re-own all of that with no contract and no merge path. A Frame inherits it and stays mergeable.

**C does not exist as a no-fork option.** `theme.css` and the tenant package change colours and the product name only. Skins change layout, but they are compiled into the app, so using one means building our own `app/`.

What each option costs:

- **A through the skin seam (recommended).** About 2,000 new lines in one new directory (OpenDots is 1,969). Edits to upstream files: one registry entry, one stylesheet import, and probably the Conversation route (241 lines) so our right panel replaces its own. Settings and Admin keep OpenBot's shell unless two more small layout files are edited. We serve our own build and proxy `/api` to the server.
- **A by reshaping the stock shell.** The same result by editing about 32 files and 5,700 lines that upstream is itself replacing (facts 6 and 7). Every upstream release then conflicts in the files we changed most.
- **B.** Either vendor about 308 files and 57,000 lines out of `app/` (39% of it) into a repo that can no longer merge upstream, or rewrite them. Either way we track an unspecified API by reading upstream's source on every release. OpenBot's own thin second client shows what the cheap version of B looks like: text only, no streaming, no cards (fact 5).

## Facts

### 1. How the shell is composed

- The router is TanStack Router with a generated file-route tree. `app/src/router.tsx:6-15`, `app/src/routeTree.gen.ts` (2,466 lines).
- There are 112 route files: 3 layout routes, 47 under `_authed/_app/`, 33 under `_authed/admin/`, 25 under `_authed/settings/`, and 4 others (`/sign`, onboarding, desktop return, a sign-in request). Counted with `find app/src/routes -name '*.tsx'`.
- Three nested layouts wrap every signed-in page:
  - Root: theme, tooltips, toasts. `app/src/routes/__root.tsx:121-135`
  - `_authed`: the session guard, then `CopilotProvider`, the Bot-look provider, hotkeys, the command palette. `app/src/routes/_authed.tsx:14-51`
  - `_authed/_app`: the visible shell. If the active skin has a `Frame` it renders `<Frame><Outlet/></Frame>`; otherwise `SidebarShell` + `AppSidebar` + `SidebarInset`. `app/src/routes/_authed/_app.tsx:13-43`
- Settings and Admin do not go through that file. Each has its own layout route with its own sidebar, so a skin `Frame` never wraps them. `app/src/routes/_authed/settings/route.tsx:10-19`, `app/src/routes/_authed/admin/route.tsx:31-40`
- `components/layout/*` is small: `SidebarShell` (persisted open state and width, 94 lines), `PageShell` (title, back link, column; 119 lines), `DetailPanel` (main pane plus a side pane that becomes a sheet when narrow; 165 lines). `app/src/components/layout/sidebar-shell.tsx:30-40`, `app/src/components/layout/page-shell.tsx:10-25`, `app/src/components/layout/detail-panel.tsx:9-36`
- The stock sidebar lists Conversations, not Bots, and opens the one live socket. `app/src/components/app-sidebar/app-sidebar.tsx:376-382`
- Pages assume the shell loosely:
  - 65 files use `PageShell`, which asks for an *optional* sidebar and draws no toggle when there is none. `app/src/components/layout/page-shell.tsx:59-63`, `app/src/components/layout/sidebar-toggle.tsx:29-31,73-75`
  - `DetailPanel` has three callers (the Conversation route, the direct Bot chat, the skills settings page).
  - All 47 `_app` pages already render inside three other frames. Pages publish `data-slot` hooks "for skins" so a frame can restyle them without depending on their classes. `app/src/skins/opendots/skin.css:215-217`
- The one tight spot: the Conversation route owns its header and its own right-hand pane. It opens OpenBot's computer panel or the Bot profile from `?watch` / `?settings` at a fixed 360 px. `app/src/routes/_authed/_app/channel/$channelId.tsx:37,90-118,120-183`

### 2. What a skin is

- A skin is a TypeScript object with up to three powers. `app/src/skins/types.ts:15-37`
  - `Frame`: "Replaces OpenBot's sidebar frame around every page under `/_authed/_app`." It must render the page as `children` and keep Settings reachable. `app/src/skins/types.ts:21-26`
  - `screens`: extra pages only this skin has, served at `/s/<key>`. "They draw OpenBot's existing data; a skin adds no endpoint." `app/src/skins/types.ts:32-36`, `app/src/routes/_authed/_app/s/$screen.tsx:9-19`
  - `botLook`: orb or face. `app/src/skins/types.ts:27-31`
- So a skin changes layout and adds components, not just colours. Colours are a separate stylesheet scoped to `:root[data-skin="<id>"]`. `app/src/skins/skins.css:1-9`
- Four skins are compiled in: `openbot` (the default, no `Frame`), `openduet`, `opendots`, `openfuse`. `app/src/skins/registry.ts:12-25`
- Selection is per person, in Settings. The id is saved on the account through `/api/settings/preferences`, cached in `localStorage`, and the account's value wins. `app/src/skins/store.ts:5-15`, `app/src/skins/account-sync.tsx:13-26`, `app/src/lib/settings/message-list.ts:123-128`
- The list of ids lives in `shared/` "because the choice is saved on the person's account, so the server has to know which values it may store". `shared/user-preferences.ts:7-18`
- OpenDots is the nearest thing to our Workbench. Its frame is a 48 px icon rail, a 220 px sidebar of Bots and recent Conversations, a top bar with breadcrumbs, and a computer pane on the right, around OpenBot's untouched pages. `app/src/skins/opendots/frame.tsx:177-269,389-541`, `app/src/skins/opendots/sidebar.tsx:170,379,643-646`, `app/src/skins/opendots/computer-pane.tsx:22-93`
- OpenDots handles the tight spot from fact 1 by deferring: on a Conversation its computer toggle drives the route's own `?watch` pane and it does not open a second one. `app/src/skins/opendots/frame.tsx:421-440,526-536`
- Size of a skin: opendots 8 files / 1,969 lines, openduet 12 / 1,856, openfuse 14 / 1,945.

### 3. What `theme.css` and the tenant package can change

- `app/src/theme.css` holds the default look as shadcn colour, radius and sidebar tokens for `:root` and `.dark`. "Only shadcn's own tokens, so a stock shadcn theme drops in." `app/src/theme.css:3-53,55-93`. A test keeps it the single place the default look is declared. `app/tests/design-tokens.test.ts:4-7`
- The tenant package is a folder of YAML the server loads: `brand.yaml`, `agents.yaml`, `channels.yaml`, `model.yaml`, `knowledge.yaml`, optional `skills.yaml`. `docs/configuration.md:1201-1216`
- Only two of its fields reach the browser, at build time: `tenant.id` and `tenant.product_name`. `scripts/generate-app-config.ts:6-12,66-71`, `app/package.json:11-13`
- `brand.yaml` may name a `theme.css`, which "may define only `:root` and `.dark` blocks, approved theme variables, and no `@import` or `url()`". `docs/configuration.md:1240-1247`. Neither example package ships one.
- Neither can add, remove or move a region. Upstream says so about renaming: "it swaps the name and logo. The visual world stays." `app/PRODUCT.md:41`

### 4. The reusable pieces

Every piece below expects the same surroundings: the React Query client exported as a singleton (91 files import it directly, `app/src/query-client.ts:3`), TanStack Router, the `@openbot/shared` workspace package (104 files), the `@/` alias, Tailwind 4 with the token stylesheets, and a same-origin session cookie. Closure sizes are from a regex walk of local imports, so they are estimates.

- **Conversation and composer.**
  - `ChannelChat({ channel, runtimeAgentId })` is the wired component. It calls CopilotKit's `useAgent({ agentId, runtimeAgentId, threadId })` and `useCopilotKit()`, and wraps the view in `CopilotChatConfigurationProvider`. `app/src/components/channels/channel-chat.tsx:509-532,1705-1723`
  - `ConversationView` is presentational: AG-UI `messages` in, `onSubmit` / `onStop` out. It renders `ChatTranscript` and `Composer`. `app/src/components/channels/conversation-view.tsx:29-56,467,531`
  - The transcript draws tool calls through CopilotKit's `useRenderToolCall` and `useRenderActivityMessage`, and the Approval card inline. `app/src/components/channels/chat-transcript.tsx:1530,1585,1639`
  - It needs `CopilotProvider` above it, which mounts twelve tool components (computer, sign-in, handoff, gallery, skills and others). `app/src/lib/copilot/provider.tsx:44-99`
  - Closure: 214 files, about 38,600 lines. The chat, view, transcript and composer files do not import the router; 6 files deeper in the closure do.
- **The Bot's computer.**
  - `ComputerViewPanel({ agentId, name })`: Screen and Terminal tabs with an activity list under them. `app/src/components/computer/computer-panel.tsx:29-107`
  - The small screen polls a screenshot; the full-size view and the terminal are WebSockets (fact 5). `app/src/components/computer/computer-view.tsx:514,1147`
  - Its render path calls no router hook and no CopilotKit hook, so it works outside a Conversation. Already reused as a right pane. `app/src/skins/opendots/computer-pane.tsx:88`. Closure: 117 files, about 16,800 lines.
- **Approvals and "needs you".**
  - `InlineApproval({ toolCallId })`, `ConversationApprovals({ threadId })`, `WaitingForYou({ agentId? })`, `NeedsYouBoard({ agentId? })`. `app/src/components/approvals/inline-approval.tsx:61,179`, `app/src/components/approvals/waiting.tsx:619`, `app/src/components/needs-you/board.tsx:35`
  - All read one inbox, `GET /api/approvals`, polled every 10 seconds, so answering in one place answers everywhere. `app/src/lib/approvals.ts:144-153,211-212`
  - `waiting.tsx` uses the router and a layout part. `app/src/components/approvals/waiting.tsx:10,13`
- **Activity.** Two different things.
  - The live list beside the screen, `ActivityLog({ computerId })`, reads a module-level store that only the mounted Conversation fills. It shows what that Conversation did, not the Bot's whole record. `app/src/components/computer/activity-log.tsx:10-13,266-270`, `app/src/components/channels/channel-chat.tsx:944`
  - The full record is `AuditPage` / `AuditTable` over `GET /api/admin/audit-events`, with a per-Bot filter. `app/src/components/manage/audit-page.tsx:156`, `app/src/lib/audit/queries.ts:38-61`, `app/src/routes/_authed/settings/_personal/activity.tsx:9-27`
- **Inline result cards (the gallery).**
  - Any file in `app/src/components/gallery/*.tsx` that exports `GALLERY` is picked up at build time and registered as a CopilotKit frontend tool. `app/src/lib/copilot/gallery-registry.ts:81-84`, `app/src/lib/copilot/gallery-tools.tsx:65,147,317`
  - Seven files ship cards today (charts, cards, table, form, decisions, quote, activity), 3,018 lines in the folder. Each call is checked against server-side grants. `app/src/lib/components/queries.ts:64,133,165`
  - A card needs no shell. This is where SEO-specific cards go, and the name is "the stable fork-facing component id". `app/src/lib/copilot/gallery-registry.ts:20`
  - The only no-fork route to a custom card: sandboxed components "authored in `/admin/playground` and published with no deployment". `README.md:391`

Taken together, the Conversation, its provider, the computer panel and the Approval views close over about 308 files and 56,800 lines, 13 of which import the router and link to the app's own routes.

### 5. How the frontend talks to the server

- **HTTP.** Every call is a relative `/api/...` fetch with `credentials: "include"`. `app/src/lib/client.ts:32-43`. Roughly 200 distinct paths in 34 route families, called by hand from 94 files.
- **Generated client.** Orval generates hooks, types, Zod schemas and MSW mocks from `api/openapi.yaml`. `app/orval.config.ts:16-45`. Only 5 non-generated files use it.
- **The spec is not complete enough to build against.** It defines three paths and six operations, Bots and reactions only. `api/openapi.yaml:40-45,93-98,147,221-227,263,323`. Conversations, Approvals, computers, Routines, Activity, memory, skills, settings and auth are all outside it.
  - One of the six is still rewritten to a legacy address in the client. `app/src/lib/api/served-today.ts:1-15,31-49`
  - CI blocks breaking changes to the spec only. The other paths have no such protection. `api/accepted-breaking-changes.md:3-6`
- **Marked not ready or reserved.**
  - The bearer-token and API-key schemes each say "The server does not accept it yet", so only the browser session cookie authenticates. `api/openapi.yaml:364-385`
  - Under `/api/v2/organizations/{id}/` these names belong to CopilotKit Intelligence and OpenBot must not use them: `channels`, `projects`, `threads`, `memories`, `api-keys`, `usage` and nine more. `api/intelligence-reserved.yaml:16-37`
  - An install has exactly one organization, `default`, "until multi-organization lands". `api/openapi.yaml:18-21`
- **Auth.** A Better Auth session cookie, checked by the `_authed` guard before anything renders. `app/src/lib/auth/client.ts:5`, `app/src/routes/_authed.tsx:14-20`, `api/openapi.yaml:355-363`
- **CopilotKit.** `runtimeUrl="/api/copilotkit"` with `credentials="include"` and no public key. `app/src/lib/copilot/provider.tsx:28-37,44-49`. `@copilotkit/react-core` is pinned at exactly 1.88.0 and moves together with the server's runtime. `app/package.json:20`, `CHANGELOG.md:230-233`
- **The browser is part of the Bot's toolchain.** The computer tools (`computer_navigate`, `computer_click`, `computer_run_command` and others) are registered in the browser with `useFrontendTool`. `app/src/lib/copilot/computer-tools.tsx:342-942`, `docs/architecture.md:36`. A client that does not register them starts runs without them.
- **Streams.** Three WebSockets, same-origin in production:
  - the live screen, `/api/computers/{id}/stream`. `app/src/components/computer/live-screen.tsx:383-384`
  - the terminal, `/api/computers/{id}/terminal`. `app/src/components/computer/live-terminal.tsx:174-176`
  - Conversation events, `/api/channels/events`. `app/src/lib/channels/use-channel-events.ts:312-328`
  - Address rules: `app/src/lib/socket-url.ts:1-16,40-55`. The small screen view polls `GET /api/computers/{id}/screenshot`. `app/src/lib/computers/screen.ts:52`
- **Serving.** The published server image serves the stock built app on its own origin. `docker-compose.yml:81,194-202`. From a checkout, `app/serve.ts` serves a build and forwards `/api`, WebSocket upgrades included. `app/serve.ts:1-14,205-210`. Separate app and API origins are supported through `TRUSTED_ORIGINS` and `OPENBOT_APP_URL`. `docs/configuration.md:423,426,570`
- **Precedent for B inside the repo.** `mobile/` is a separate Expo client, 20 source files and 2,833 lines. It calls the same unspecified `/api/...` paths with the cookie attached by hand, has no CopilotKit dependency, and its Conversation screen loads history, posts a message and refreshes. A tool call shows as the text "The Bot is using a tool." `mobile/src/api.ts:2-19`, `mobile/package.json:12-33`, `mobile/app/conversation.tsx:49-73,128-132`

### 6. Maintaining a fork

- Forking is the stated intent: "You can fork, modify, and white-label it." `README.md:107`. "OpenBot exists to be forked." `app/src/routes/_authed/_app/bot.tsx:50`
- `docs/upstream-upgrades.md` describes an opt-in workflow, **Propose upstream upgrade**. Given an exact upstream release tag and commit, it merges that release into the fork's `main` in an isolated clone, runs the candidate's own CI `static` and `test` jobs, and opens a pull request. It never merges, releases or deploys. `docs/upstream-upgrades.md:84-93`, `.github/workflows/upstream-upgrade.yml:1-3`, `scripts/integrate-upstream.ts`
- Conflicts are left for a person: "Conflicts keep the candidate's conflict markers", no pull request is opened, and "No automatic migration-history rewrites or codemods are applied." `docs/upstream-upgrades.md:86,113`
- A modified checkout is classified `forked`, and a forked install must integrate upstream source before its artifacts can be replaced. `shared/release-manifest.ts:298-301,333-343`
- The doc assumes a fork that publishes its own images ("A fork publishes its own source and images"). `docs/upstream-upgrades.md:7`
- Cadence, as far as the clone shows it:
  - `CHANGELOG.md` has 28 tagged versions from 0.0.1 to 1.0.3 plus `Unreleased`, in 8,406 lines. No heading carries a date.
  - Update-feed fixtures date 0.3.1 at 2026-11-06 and 0.3.2 at 2026-11-07. `desktop/src-tauri/test-fixtures/app-update/latest-0.3.1.json:4`, `desktop/src-tauri/test-fixtures/app-update/latest-0.3.2-with-windows.json:4`. The 1.0.3 notes cite a package released 2026-11-12, and the clone's head is dated 2026-11-14 at pull request #1599. `CHANGELOG.md:258-264`. That is seven releases (0.3.2 to 1.0.3) in about a week.
  - `Unreleased` is already 249 lines and regroups the main menu, Settings and Admin: "several pages have new names". `CHANGELOG.md:32,99-111`
  - Skins arrived in 0.3.0 and are named in five of the ten changelog sections from there on. `CHANGELOG.md:3416-3422,1157-1160,540,260,237,54-59`
  - The stock look moves more than the skins do. 0.3.0 gave it "a new look", 1.0.0 made the sidebar frosted glass and changed the typeface, and 1.0.1 took both back. `CHANGELOG.md:3397,1155-1157,935-936`
- Upstream is redesigning exactly what "reshape the shell" would edit: "The skin system stays. This redesign replaces the default `openbot` skin. The other skins stay as alternates." `app/PRODUCT.md:34`

### 7. Size

- `app/src`: 788 files; 785 TypeScript and CSS files, 139,620 lines. Of those, 45 files and 5,408 lines are generated.
  - `components/` 329 files / 82,893 lines; `lib/` 295 / 36,179; `routes/` 112 / 11,201; `skins/` 40 / 6,148.
- `app/tests`: 508 files, 118,631 lines. 19 test files name the stock shell files; 14 name skins.
- Files a stock-shell reshape would change, about 32 files and 5,700 lines:
  - `routes/_authed/_app.tsx` (43 lines)
  - `components/layout/*` (11 files, 1,136)
  - `components/app-sidebar/*` (6 files, 1,015)
  - `components/ui/sidebar.tsx` (889) and `lib/sidebar.ts` (83)
  - `lib/navigation.ts` (177) and `components/navigation/command-menu.tsx` (476)
  - the Settings and Admin layouts and sidebars (4 files, 393)
  - the six routes that draw their own header band: `channel/$channelId` 241, `channel/new` 473, `bot` 263, `index` 367, `start.$startId` 123, `group/$channelId` 43
- Files the skin route would change: a new directory of about the size of opendots (8 files, 1,969 lines), `skins/registry.ts`, `skins/skins.css`, and most likely `channel/$channelId.tsx`.

### 8. Is there a Site above Bots and Conversations?

No. Nothing in the app or the API groups Bots or Conversations under a site, project or folder.

- A Conversation (`channel`) carries an id, a name, its Bots and a thread id. No parent, no tag. `app/src/lib/channels/queries.ts:11-36,68-106`
- The specified Bot fields are name, title, role description, visibility, pinned, hidden and ownership. No grouping field. `api/openapi.yaml:455-577`
- OpenDots dropped its template's grouping for this reason: "Left out because OpenBot has nothing behind them: Spaces and their pages". `app/src/skins/opendots/index.ts:7-8`
- The closest existing concepts, none of which fits:
  - **Organization.** One per install, fixed at `default`. `api/openapi.yaml:18-21`, `app/src/lib/api/request.ts:25-31`
  - **`tenant.id`.** One per Instance, read at build and start-up. This is the Client, not a Site. `docs/configuration.md:1226-1227`
  - **`projects`.** A name reserved for CopilotKit Intelligence. `api/intelligence-reserved.yaml:28`
  - **Package `channels.yaml` with `allowed_groups`.** "validated and stored, and nothing reads it", and package-declared channels "are unreachable rather than open". `docs/configuration.md:1358-1366`
  - **Pinned.** A per-person flag on Bots and Conversations. An ordering, not a grouping. `app/src/lib/channels/queries.ts:83-84`
- Consequence: a Site has to be something we own. A skin "adds no endpoint" (`app/src/skins/types.ts:33-34`) and the server cannot be changed, so the Site-to-Bot mapping needs its own home: a naming or id convention on Bots, or a small store of ours beside the server.

## How the six regions map onto what exists

1. **Icon rail.** OpenDots' `Rail`, fed by the shared destination list. `app/src/skins/opendots/frame.tsx:177-269`, `app/src/lib/navigation.ts:87-139,170`
2. **Sidebar.** OpenDots' Bots list is reusable. The Site switcher is new and has no backing data (fact 8).
3. **Tabs.** New. Routines has a ready list, `RoutinesList({ agentId, embedded })`. `app/src/components/routines/routines-list.tsx:244-252`. Deliverables has nothing behind it: no match for "deliverable" or "artifact" anywhere in `app/src`.
4. **Conversation.** `ChannelChat`, unchanged, arriving as the Frame's `children`.
5. **Right panel.** `ComputerViewPanel`, `WaitingForYou` and `ConversationApprovals` exist. They collide with the Conversation route's own 360 px pane (fact 1).
6. **Status bar.** New. Nothing like it exists.

## Open risks and what could not be verified

- **A new skin id may not survive the server.** The server validates the saved skin and is closed, so a fifth id might be refused and the account's value would then override ours. Giving the default `openbot` entry our `Frame` avoids stored preferences altogether, but it is the entry upstream is redesigning. Needs a test against a running server. `shared/user-preferences.ts:10-11`, `app/src/skins/account-sync.tsx:13-26`
- **Serving our own build.** Whether the published server image can serve a build other than the one baked in (`APP_DIST_DIR` is "Set inside the container image", `docs/configuration.md:88`) was not verified. The safe assumption is our own static host that forwards `/api` and WebSocket upgrades, as `app/serve.ts` does.
- **Fork tooling without a server directory.** Release validation reads `server/drizzle/meta/_journal.json` (`scripts/release-manifest.ts:477`), which is not in the clone. Whether the upgrade workflow runs cleanly for a fork that changes only `app/` and keeps upstream's images is untested.
- **How much `app/` changes per release.** The clone has one commit, so no diff between tags was possible. The cadence above comes from the changelog and two fixture dates. A full clone would give `git diff --stat vA..vB -- app/`.
- **The tenant package's `theme.css`.** The list of "approved theme variables" is not in the repo, and no code in `app/` or the build script loads that file into the browser. Whether the server injects it is unknown.
- **The unspecified API will move.** The client still rewrites one specified operation to a legacy path "until it answers them at /api/v2", and `channels` and `threads` are reserved names under v2. Expect the Conversation endpoints to change address. Under A this arrives as an upstream merge; under B it is ours to find. `app/src/lib/api/served-today.ts:1-15`
- **Licences.** `app/` and `shared/` are MIT. CopilotKit Intelligence is licensed separately and required. `README.md:107-113`. Whether white-labelling changes the Intelligence terms was not checked.
- **Closure sizes** are regex estimates over local imports. Type-only imports count, npm packages and `@openbot/shared` do not.
- **Not in question, noted for the map.** The Site switcher and the Deliverables tab both need data the server does not hold. That is a larger open item than the choice of base.
