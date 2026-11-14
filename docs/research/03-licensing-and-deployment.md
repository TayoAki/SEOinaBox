# 03 · Licensing and deployment: what selling a product built on OpenBot requires

Researched 2026-11-14. Read-only: nothing was run, installed, signed up for or submitted. This is fact-finding, not legal advice.

How sources are cited:

- `path:line` is CopilotKit/OpenBot `main` at commit `1727413` (2026-11-14). `path:line @v1.0.3` is the release tag `v1.0.3` (commit `8e44272`, 2026-11-13).
- Web quotes are exact sentences taken from the page text. Every URL was fetched on 2026-11-14.
- "Registry" means an anonymous read of an image's config from `ghcr.io/copilotkit/*` on 2026-11-14.
- "Silent" means no public page I found answers the point.

## The questions

1. Licensing for resale: what is MIT, what covers the server images and Intelligence, and what do CopilotKit's pages say about white-labelling, hosting for third parties and reselling?
2. Editions and tenancy: personal versus team, and can one deployment safely serve several client organisations?
3. Deployment requirements: what must run, with what privileges and sizes, and what can the all-in-one image do?
4. Hosting fit: Railway, a single VM with Docker Compose, managed Kubernetes.
5. Cost drivers per client deployment, from primary-source figures only.

## Short answers

**1. Licensing.** Nothing public grants a right to resell OpenBot or to host it for third parties. The MIT licence covers the code in the repository and lets us fork, rebrand and sell that code. It does not cover three things the product needs:

- **CopilotKit Intelligence.** OpenBot will not start without it, and it is proprietary. Every published Intelligence term limits use to non-production (the free local licence) or to the customer's own internal business (the self-service agreement), and both forbid resale and sublicensing. The OpenBot for Business licence text is not published; the pricing page says it "covers commercial use across your organization".
- **The names and logos.** A separate trademark policy applies. A shared, changed version needs its own name and logo.
- **The backend from now on.** Through tag v1.0.3 the server, supervisor, computer, worker and Helm chart source were in the repository under MIT. They were removed from `main` after that tag, and the newest images ship compiled binaries with no licence label. What licence covers those images is silent.

Selling this needs a signed agreement with CopilotKit that names resale or hosting for clients.

**2. Editions and tenancy.** The personal edition is one person, no sign-in, one shared computer. The team edition adds sign-in, admin and user roles, SCIM groups, shared Bots, a computer per person, and admin policy and audit. The design assumes one organisation per deployment: the API spec says so, the data model has no tenant column, and administrators, public Bots, the admin vault, policy and the audit trail are deployment-wide. Putting several clients in one deployment is not supported by the docs. What the docs do support is one deployment per client. CopilotKit's website advertises multi-tenancy for OpenBot for Business; the repository does not show it. Fleet provisioning, cross-client administration, billing and per-client branding would be our own build.

**3. Deployment.** A deployment needs the OpenBot server and app, PostgreSQL, CopilotKit Intelligence, at least one Bot computer (Chromium), a scheduled sweep for routines and one for attachments, a model key, and sign-in plus TLS once anyone else can reach it. A computer per person needs either the supervisor holding the Docker socket, or Kubernetes with the agent-sandbox controller; gVisor is optional. The all-in-one image serves several signed-in users but gives every Bot of every person one shared computer, and schedules nothing. Managed (CopilotKit-hosted) Intelligence is technically supported and CopilotKit's own deployment guide allows it for a company deployment; nothing says production must self-host. Which licence permits either for client-facing use is the open point.

**4. Hosting fit.**

- **Railway:** workable only as the all-in-one image plus Railway PostgreSQL plus managed Intelligence, one project per client. No computer per person, because Railway containers are non-privileged and there is no Docker socket. Fits a demo or a small client who accepts one shared computer.
- **Single VM with Docker Compose:** workable and gives a computer per person through the supervisor. Intelligence must be managed, since self-hosted Intelligence is documented only for Kubernetes and ECS/Fargate. Compose is presented as the source and laptop path, so TLS, backups and upgrades are ours.
- **Managed Kubernetes (EKS/GKE/AKS):** the documented production target. One Helm release per namespace, optional sandbox computers, bundled self-hosted Intelligence. Largest footprint, and it needs a trial or production Intelligence licence.

**5. Cost drivers.** OpenBot for Business is "Up to $29 / user / month" per active user, plus 5% of model spend above $5,000 a month per organisation; Annual is custom. Model spend goes direct to the provider, and Automatic Learning spends the same key. Compute is at least 2 GB and 1 vCPU for the one-container shape, about 3.25 vCPU of requests for the vanilla Kubernetes install, and 500m CPU and 2 Gi requested per person's computer in sandbox mode.

## Evidence

### 1. Licensing for resale

**What is MIT**

- The root licence is MIT, "Copyright (c) 2026 CopilotKit", with the usual grant to "use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies". `LICENSE:1-21`
- It is the only licence file. A search of the tree for `LICENSE*`, `NOTICE*`, `COPYING*`, `EULA*` and third-party files finds only `LICENSE` (the other hits are source files named `notice`). Package manifests say `"license": "MIT"`. `package.json:4`, `app/package.json:3`, `shared/package.json:3`, `extension/package.json:3`
- The README states the scope as the "application template" and the "application source". `README.md:107`, `README.md:628`
- Vendored brand marks are third parties' trademarks, "used to identify that product in a picker and for nothing else". `shared/marks/README.md`
- There is no third-party notices file, so dependency licences are not listed anywhere in the repository (not audited here).

**The backend source left the repository after v1.0.3**

- On `main` there is no `server/`, `supervisor/`, `agent-computer/`, `worker/` or `charts/` directory. The top-level entries removed between `v1.0.3` and `main` are `Dockerfile agent-bot agent-computer agent-langgraph charts docker server spire supervisor toolkit worker` (`git ls-tree` on both refs).
- At `v1.0.3` those directories exist under the same root MIT licence, and each manifest says MIT: `server/package.json`, `supervisor/package.json`, `agent-computer/package.json`, `worker/package.json`, `toolkit/package.json` `@v1.0.3`. `server/` holds 1,234 files and `charts/` 39.
- The change is recorded as unreleased: "The server runs from the published OpenBot images." and "there is no chart in the checkout to build from." `CHANGELOG.md:9`, `CHANGELOG.md:222-228`
- The image workflow builds only the agent harnesses here; "The server, computer and supervisor images, and the all-in-one `openbot` image, are the published OpenBot images". `.github/workflows/images.yml:8-11`, `backend-images.env:1-20`
- Registry, tag `v1.0.3`: `openbot-server` runs `bun src/production-entry.ts`, carries the TypeScript source, and its `package.json` says MIT. No image has an OpenBot licence label; the only `org.opencontainers.image.licenses` value is `NOASSERTION`, inherited from the Bun base image.
- Registry, tag `main` (built 2026-11-14): `openbot-server` runs a compiled binary, `/app/bin/openbot server`; the supervisor and computer images likewise. The only label is `org.opencontainers.image.source=https://github.com/CopilotKit/OpenBot`. The build is stamped with public commit `6f9f77e`, whose tree has no `server/`.
- The docs on `main` still cite `server/src/...` paths that no longer exist there, for example `docs/architecture.md:81` and `docs/architecture.md:191`.
- Silent: the licence of the compiled images, whether the server source will be published again, and whether images may be re-hosted or redistributed.

**CopilotKit Intelligence is required and separately licensed**

- "It uses **CopilotKit Intelligence**, which is separately licensed and can run entirely on your own machine or infrastructure." `README.md:107`
- The server refuses to start without the Intelligence URL, gateway URL and key; "There is no degraded mode." `README.md:435-441`, `.env.example:122-124`
- The licence table: laptop "Requires a free local Intelligence license"; Kubernetes evaluation "Requires a free Intelligence trial license"; Kubernetes / production "Requires a production Intelligence license beyond evaluation. OpenBot for Business includes Intelligence for use with OpenBot." `README.md:109-113`
- "Running it in production on your own infrastructure is an Enterprise Intelligence Platform feature deployed by Helm chart". `.env.example:129-130`
- "The OpenBot application is available under the MIT license. CopilotKit Intelligence is a required service with separate terms, whether managed or self-hosted. Self-hosting it requires a valid license for that deployment." https://open.bot/resources
- "The application’s MIT license does not change the terms for Intelligence, model providers, or connected tools. Our team will confirm the services and support included in your business agreement." https://open.bot/resources
- "Self-hosting is available on the Team self-hosted plan or a custom Enterprise plan for VPC, on-prem, and data-residency deployments." https://docs.copilotkit.ai/intelligence/self-hosting
- Without a valid licence a self-hosted Intelligence "refuses memory and the other licensed features while conversations carry on". `docs/configuration.md:46-49`

**The free local licence (Developer Edition) cannot be used to sell or host**

- The desktop's local-terms step names this agreement. `desktop/src-tauri/src/local_eval.rs:10015`
- "PRODUCTION USE REQUIRES A PAID COPILOTKIT INTELLIGENCE LICENSE UNDER A SIGNED AGREEMENT." https://www.copilotkit.ai/legal/developer-edition (Last Updated: October 1, 2026)
- Production Use includes any use that "(b) supports any internal business operation, customer-facing service, or revenue-generating activity; (c) is accessible from the public internet or by Customer's customers, partners, or end users". Same page, section 2.
- The customer will not "sell, resell, rent, lease, lend, sublicense, distribute, or otherwise make the Developer Edition available to any third party, including by publishing, mirroring, or re-hosting the container image in any public or third-party registry, or by offering it as a hosted or managed service". Same page, section 5(b).
- "Use it to try Intelligence on one machine. It is not a production installation." https://docs.copilotkit.ai/intelligence/self-hosting-local

**Managed Intelligence (self-service agreement) is for internal use and bars resale**

- "It governs your access to and use of CopilotKit's self-service, online, and cloud-hosted offerings, including the self-service tiers of CopilotKit Enterprise". https://www.copilotkit.ai/self-service-agreement (Last Updated: August 10, 2026)
- The grant is "a limited, non-exclusive, non-transferable, non-sublicensable, revocable right to access and use the Services for Customer's internal business purposes during the Subscription Term". Same page, section 8.2.
- The customer will not "sell, resell, sublicense, or otherwise commercially exploit the Services except as permitted here". Same page, section 5(b).
- "End Users are users of Customer's products, not of the Services." Same page, section 2.6. An Authorized User is "an individual within Customer's workforce". Section 2.1.
- A Signed Agreement (Master Services Agreement and Order Form) "prevails over this Agreement to the extent of any conflict". Same page, section 1.
- The general terms add: you may not "Use the Services to build similar or competitive services." https://www.copilotkit.ai/terms-of-service

**OpenBot for Business**

- "Personal OpenBot is for development and experimentation. OpenBot for Business covers commercial use across your organization, with deployment and support agreed with our team." https://open.bot/pricing
- "The OpenBot application source is MIT-licensed. The service terms for the complete offering are separate from that source license." https://open.bot/pricing
- "Your OpenBot for Business license covers CopilotKit Intelligence to run OpenBot for Business. Building agents into your own product requires a separate Intelligence license." https://open.bot/pricing
- "OpenBot for Business is a separate product with its own license, usually bought by IT to give employees an AI assistant." https://www.copilotkit.ai/pricing
- The service terms themselves are not linked from the pricing page. The only legal pages in either sitemap are trademarks, developer-edition, support-terms, the self-service agreement and the terms of service.

**White-labelling**

- "Use the MIT-licensed template as-is, white-label it, or customize it deeply." `README.md:70`. "OpenBot is a template to make your own and white-label, not a hosted product somebody signs up for." `prompt.txt:25`
- "Use it as-is, white-label it, or deeply customize it." https://open.bot
- "it is built to be white-labelled: CopilotKit's engineers can rename it, reskin it and customize how it works." https://www.copilotkit.ai/blog/openbot-for-business-the-controls-it-asks-for-before-an-ai-agent-goes-live
- The trademark policy limits the names: "The CopilotKit OpenBot code uses the MIT License. That license lets you use, copy, change and share the code. It does not give you the right to use our names or logos." https://open.bot/legal/trademarks (Last updated: 03Nov2026)
- "You can share a changed version under its own name, for example "Acme Agent (based on OpenBot)", if you give it its own name and logo, take our logos out of it, and say it is based on OpenBot." Same page.
- "A name for a hosted service or a paid software product that starts with OpenBot or CopilotKit, or that could make people think we run it, needs our permission". Same page.
- "Do not suggest that CopilotKit made, sponsors, certifies, partners with or approves a product, service, company or event. Only a written agreement with us allows it." Same page.
- "Always keep our copyright and license notices, as the MIT License requires." Same page.
- In the product: `tenant.product_name` in `brand.yaml` and an optional `theme.css` limited to approved variables. `docs/configuration.md:1218-1247`. "OpenBot" and "OpenBot for Business" carry a trademark sign in the app, and the app has "OpenBot for Business" entries and a self-hosting banner. `app/src/components/edition/brand-name.tsx:5`, `docs/configuration.md:111-123`

**Hosting for third parties and reselling**

- Silent. No page on open.bot, copilotkit.ai or docs.copilotkit.ai, and no file in the repository, mentions resellers, partners, agencies, OEM, managed service providers, or a customer hosting OpenBot for other organisations. A repository-wide search for those words finds only the white-label lines above.
- The nearest statements are CopilotKit hosting it itself: "Self-hosted by default, or hosted by us in early access" (https://open.bot/pricing) and "Deploy in your own VPC on AWS, Azure or GCP, on-prem, or on OpenBot Cloud." `app/src/components/edition/team-features-dialog.tsx:42-43`

### 2. Editions and tenancy

**Personal versus team**

- "`OPENBOT_EDITION=personal` runs OpenBot for one person on their own machine: nobody signs in, there is one computer, and the deployment starts with a single Bot". Unset or `team` "is OpenBot as an organization runs it". `docs/personal-edition.md:3-5`
- Personal requires `OPENBOT_SINGLE_USER=true`, no identity provider, and one shared computer with no supervisor. `docs/personal-edition.md:99-103`
- Off in personal: "Slack and Teams, Team Bots, Isolated Bots" and "SSO, action recording, model and MCP allowlists, skill sharing". `docs/personal-edition.md:116-120`
- The server answers `404` for the team-only APIs: "people, identity providers, enterprise controls ..., the admin learning and check-in settings, and SCIM and SSO registration". `docs/personal-edition.md:249-253`
- "The personal edition has no memory at all". `docs/architecture.md:371`
- Team adds sign-in with Google, Microsoft or Okta, plus SAML or OIDC "routed by email domain" (`README.md:394`, `README.md:531-533`); the People screen (`README.md:395`); SCIM, whose directory groups "become their OpenBot groups, which per-group capability switches and network policies read" (`docs/configuration.md:632-649`); Team Bots published "to everybody signed in or to named people and groups" (`docs/coworkers.md:63-65`); a computer per person (`README.md:383`).
- Roles are two: `pgEnum("role", ["admin", "user"])`. `server/src/db/schema/core.ts:29 @v1.0.3`

**One organisation per deployment**

- "Until multi-organization lands, an OpenBot install has exactly one organization, and organizationId is that organization. Clients address it as `default`; the server treats any other value as an organization that does not exist." `api/openapi.yaml:18-21`
- The Helm install "creates OpenBot's organization and project" in Intelligence, one of each. `charts/openbot/README.md:74-75 @v1.0.3`
- The only `tenant_id` column is on `deployment_packages`. `users`, `agents`, `credentials` and `audit_events` have no tenant or organisation column. `server/src/db/schema/core.ts:64,263,425,448 @v1.0.3`
- `tenant.id` in `brand.yaml` "owns the Bots and channels the package seeds, and names the deployment when `DEPLOYMENT_ID` is unset". `docs/configuration.md:1226-1227`. It is a package identifier, not a customer boundary.
- `DEPLOYMENT_ID` "Names this deployment inside a shared Intelligence project". It separates two deployments, not two clients inside one. `docs/configuration.md:71`, `docs/configuration.md:1194-1197`
- A `public` Bot is visible to "Everyone in the deployment"; a `private` one to "Owner and administrators". `docs/coworkers.md:56-59`
- The admin vault is "write-only encrypted credentials for the deployment". `README.md:363`. Audit and policy are deployment-wide. `README.md:396`, `docs/architecture.md:78-81`
- Capability scopes are `organization`, `role` and `group`, and a group "can only widen". `server/src/db/schema/admin.ts:29-33 @v1.0.3`
- Channel `allowed_groups` is not enforced: "Treat `allowed_groups` as a declaration, not as a control that is running." `docs/architecture.md:188-193`
- A shared computer "is fine for a deployment where one team trusts its own Bots and is not fine as a boundary between tenants". `docs/deployment.md:69-72`. On it, "one Bot's shell can read another's files". `prompt.txt:277-278`
- Memory is per person: "one person's Bot never sees another person's memory". `docs/architecture.md:384-386`

**What the docs support instead: one deployment per client**

- "install one OpenBot release per namespace". `prompt.txt:265`
- "To run two deployments on one Docker host, give the second one its own `COMPOSE_PROJECT_NAME`, `COMPUTER_NAMESPACE` and `COMPUTER_IMAGE`." `docs/configuration.md:1190-1192`
- "Each deployment keeps its own Intelligence project credential." `README.md:564`
- A team deployment can delegate sign-in to another OpenBot with `OPENBOT_ORGANIZATION_AUTH_URL`. `README.md:554-561`. This could give our own staff one sign-in across client deployments; the docs describe it for one company, so that use is our inference.
- Two single-user deployments on one Composio or Arcade key once shared connections, which is why each now gets its own broker identity. `docs/configuration.md:474-482`. Separate broker projects per client avoid the question.

**The website says more than the repository shows**

- "Multi-tenancy" and "Isolate each customer’s data in its own tenant, with its own settings and access controls, while super-admins oversee them all." https://open.bot/business (same text on https://open.bot)
- "Set daily, weekly, and monthly token limits per tenant and per person, and see spend for every tenant." https://open.bot/business
- The repository has no "super-admin", and no tenant in the API spec. Whether this is shipped, planned, or only in OpenBot for Business or OpenBot Cloud is silent.

**Our own build** (not in the docs): provisioning and upgrading a fleet of per-client deployments; any view or admin across clients; client billing and metering; per-client branding beyond a name and theme variables; any isolation between clients inside one deployment.

### 3. Deployment requirements

**What must run**

- Services: `app` 3010, `server` 3001, `agent-computer` 4100, `supervisor` 4500/4300, `worker`, PostgreSQL 5432, and Intelligence, listed as "external". `README.md:475-485`
- PostgreSQL is `pgvector/pgvector:pg17` in Compose; with a managed database the role must be able to create the `vector` extension. `docker-compose.yml:3`, `docs/deployment.md:57-63`
- Sign-in is required on anything shared: single-user mode "refuses to start" on a public address. `docs/deployment.md:154-165`. "Put TLS in front of it." `docs/deployment.md:167`
- Routines need a scheduler: the Helm chart runs a CronJob "every five minutes"; Compose runs a `worker`. `prompt.txt:287-288`, `docker-compose.yml:227-232`
- The Helm install is "OpenBot (two API replicas, one shared Bot computer, and the scheduled Jobs) and CopilotKit Intelligence ..., self-hosted on its own PostgreSQL, Redis and object storage". `prompt.txt:156-159`
- Self-hosted Intelligence needs "Kubernetes ≥ 1.28", "PostgreSQL ≥ 14", "Redis ≥ 7" and "An OIDC identity provider", plus "A valid CopilotKit Intelligence license key". An ECS/Fargate CloudFormation bundle is the documented alternative. https://docs.copilotkit.ai/intelligence/self-hosting

**Privileges**

- "The Docker socket is root-equivalent on the host, so only the supervisor holds it and only exposes Bot-level verbs." `docker-compose.yml:334-335`. It is mounted read-only with SELinux labelling disabled. `docker-compose.yml:394-397`
- The supervisor "needs a Docker socket, which no serverless container platform permits". `docs/deployment.md:67-68`
- No privileged containers are requested. The shared computer adds `NET_RAW`. `docker-compose.yml:260-261`. Supervisor-made computers run with `CapDrop: ["ALL"]`, a short capability list and `no-new-privileges`. `supervisor/src/docker.ts:567-571 @v1.0.3`
- gVisor is optional: "Set to runsc to run every computer under gVisor, when the host has it. Left unset each computer is an ordinary container and shares the host kernel." `docker-compose.yml:367-368`
- On Kubernetes, `computers.mode: sandbox` gives "A computer per person (and one per isolated Bot)" and needs "The `agent-sandbox` controller in the cluster". `charts/openbot/README.md:730-732 @v1.0.3`. `runtimeClassName` is empty by default because "GKE has managed gVisor, AKS offers Kata and gVisor, and on EKS it is bring your own node configuration". `charts/openbot/values.yaml:388-391 @v1.0.3`
- "A Bot can run anything on its own person's computer, root included". `docs/architecture.md:87`. "gVisor and a computer per Bot remain the actual boundary." `charts/openbot/values.yaml:405 @v1.0.3`
- The cluster needs amd64 nodes and a default StorageClass. `README.md:123`, `prompt.txt:164-168`

**Minimum sizes**

- One container: memory minimum 2 GB, recommended 4 GB; vCPU minimum 1, recommended 2; disk minimum 4 GB, recommended 8 GB. "Every additional concurrent page is roughly another 100 to 200 MB". `docs/deployment.md:117-126`
- Kubernetes vanilla install: "About 3.25 vCPU of requests while it runs, plus about 0.4 vCPU while the install Jobs run." Two `t3.large` nodes "come out short". `prompt.txt:169-171`
- A person's computer in the chart: requests `cpu: 500m`, `memory: 2Gi`, limit `memory: 8Gi`, with 10Gi each for profiles and workspace. "Chromium is roughly 200-500MB per browser ... and up to 2GB with real pages open." `charts/openbot/values.yaml:489-507 @v1.0.3`
- Local Intelligence evaluation: "at least 4 CPUs, 12 GiB RAM, and 40 GiB free storage for each stack". https://docs.copilotkit.ai/intelligence/self-hosting-local

**The all-in-one image `ghcr.io/copilotkit/openbot`**

- In it: "the built app, the API, and Chromium. One port, 3001", and PostgreSQL if `EMBEDDED_POSTGRES=on`. `docs/deployment.md:39-45`
- Several users: yes, with an identity provider configured. `docs/deployment.md:138`
- A computer per Bot or person: no. "Without it, every Bot shares the one computer". `docs/deployment.md:67-72`. Replicas do not help: "every replica shares the one browser inside it". `docs/deployment.md:225-227`
- No routines schedule and no attachment sweep; both need "something outside the container". `docs/deployment.md:74-98`
- No managed LangGraph Bot; leave `MANAGED_AGENT_AG_UI_URL` unset. `docs/deployment.md:148-152`
- Kubernetes "is the only one that gives each person (or an Isolated Bot) a computer of its own, runs the routines schedule without something outside the container, and scales the API past a single replica". `docs/deployment.md:243-245`
- Bot files and browser profiles are two paths, `/workspace` and `/profiles`. `Dockerfile:266-269 @v1.0.3`

**Managed or self-hosted Intelligence**

- The source setup "uses managed Intelligence by default". `README.md:163`. The Helm chart "installs Intelligence by default" and takes `intelligence.enabled: false` to point at managed Intelligence. `README.md:121`, `prompt.txt:270-274`
- "A company deployment can use managed Intelligence or host it alongside OpenBot in your own infrastructure." https://open.bot/resources
- "Pay as you go and Annual support hosted and self-hosted deployments." https://www.copilotkit.ai/pricing
- With managed Intelligence, threads, memory, Learning and recordings are held by CopilotKit, not in our infrastructure. `docs/architecture.md:23`, `docs/configuration.md:531-535`

### 4. Hosting fit

**(a) Railway**

- OpenBot's own note: "Railway, Render, Fly.io. All run this image directly and all provision PostgreSQL in a click". `docs/deployment.md:253-254`
- "Because Railway containers are non-privileged, GitHub Workflows that build-and-then-mount containers on the same host (i.e. Docker-in-Docker) will fail." https://docs.railway.com/guides/github-actions-runners. So the supervisor cannot run there, and each client gets one shared computer.
- "Each service can only have a single volume" and "Replicas cannot be used with volumes". https://docs.railway.com/volumes/reference. The image keeps Bot files and profiles in two paths, so with one volume one of them is lost on redeploy unless we change the layout (inference).
- Volume defaults: "Hobby plans: **5GB**", "Pro plan: **50GB**". Same page.
- "Private networking enables secure communication between services within your Railway project without exposing traffic to the public internet." https://docs.railway.com/networking/private-networking. It is "only available at **runtime**, not during the build phase". https://docs.railway.com/networking/private-networking/how-it-works
- Cron services can run the routines sweep: "Scheduled runs must be at least 5 minutes apart in UTC." and the service must exit when done. https://docs.railway.com/cron-jobs
- Compose import is partial: "we don't support every possible Compose config just yet". https://docs.railway.com/builds/dockerfiles
- Railway Sandboxes are "isolated Linux VMs you create on demand, run commands in, and destroy". https://docs.railway.com/sandboxes. OpenBot documents no provider for them; its computer modes are `shared`, `sandbox` (Kubernetes) and `external` (one shared URL). `prompt.txt:276-285`. Using them would be our own build.
- Self-hosted Intelligence has no documented Railway path, so Intelligence would be managed.

**(b) Single VM with Docker Compose**

- Compose defines PostgreSQL, migrations, server, worker, the shared computer and the supervisor, all from published images. `docker-compose.yml:2-339`, `docs/configuration.md:1131-1144`
- With `COMPUTER_SUPERVISOR_URL` "each person gets a computer container of their own". `docs/architecture.md:91`. gVisor can be installed on a VM we control.
- Ports bind to loopback, so a TLS reverse proxy is ours to add. `docker-compose.yml:193-204`
- The docs frame this as the source path: the "three supported paths" are the one-container image, the clone, and the Helm chart. `prompt.txt:9-15`. The clone "is for running it locally and for changing the product". `prompt.txt:26-27`
- Several client stacks on one host are documented (`docs/configuration.md:1190-1197`), but computers share the host kernel unless gVisor is on, and a Bot has a root shell. One VM per client, or gVisor, is the safer shape (inference).
- Intelligence would be managed, or self-hosted elsewhere on Kubernetes or ECS/Fargate.

**(c) Managed Kubernetes**

- "Deploy OpenBot and CopilotKit Intelligence together on **AWS EKS, Azure AKS, Google GKE, or your own Kubernetes cluster**." `README.md:121`
- The chart ships values for EKS, EKS sandbox, GKE and AKS. `charts/openbot/README.md:456-468 @v1.0.3`
- "The bundled one is not production-grade: no backups, failover or point-in-time recovery. A real deployment points at RDS, Cloud SQL, Azure Database or its own server". `prompt.txt:239-240`
- The published image for releases is `linux/amd64` only. `charts/openbot/README.md:121 @v1.0.3`
- The AWS Marketplace listing "covers deployment to Amazon EKS". https://www.copilotkit.ai/pricing

### 5. Cost drivers per client deployment

**OpenBot for Business** (https://open.bot/pricing)

- "OpenBot is free for personal projects. OpenBot for Business™ costs up to $29 per active user per month." Pay as you go is "A per-user rate that falls as your team grows", by credit card, "with no commitment".
- "A 5% charge on model spend above $5,000 per month, per organization, applies to both pay as you go and Annual."
- Active user: "A named person with at least one interaction with OpenBot for Business in the billing month. Only production usage counts. Each person counts once. Bots, agents, API calls, and guests are never counted."
- "Unlimited Bots, with no per-bot, per-agent, or per-interaction charge."
- Annual is "Custom pricing" and is where "SSO and role-based access control", "Offline and air-gapped deployment", "Custom contract terms" and "24/7 production support" are listed.
- "Hosted deployments are available in early access at the same plan price."

**Intelligence bought on its own** (https://www.copilotkit.ai/pricing)

- Free: "Developer Edition: build and evaluate on your own machine. Free for non-production use with a renewable 30-day license." Free hosted has "1,000 interactions per organization each month".
- Pay as you go: "$49 per organization with 10,000 interactions included, then graduated rates for additional interactions. Hosted and self-hosted have the same price."
- Self-hosted pay as you go "Requires a connected deployment that reports usage counts for billing." Offline needs Annual.
- "Development, staging, test and demo usage is free. Paid usage covers your production deployments."

**Model spend**

- "You pay your model providers directly. We do not resell inference or mark up tokens." https://open.bot/pricing
- Automatic Learning "spends their model key" on `openai/gpt-5.5` or `anthropic/claude-sonnet-5-5`. `prompt.txt:266-267`
- Routines have "A 15-minute floor and a cap of 20 enabled routines", and "ten failures in a row switch a routine off rather than burn model spend forever". `README.md:400`
- A Bot handing work to another Bot "is a whole agent turn at the other end". `docs/architecture.md:281-285`

**Compute**

- The sizes in section 3. Browsers dominate memory: up to 8 at once per computer by default, closed after 30 idle minutes. `docs/configuration.md:837-839`
- Sandbox computers are "suspended when idle and resumed". `prompt.txt:279-280`
- Staged attachments can reach "256 MiB of staged blobs per person". `docs/deployment.md:100-103`
- Railway rates, if used: RAM "$10 / GB / month", CPU "$20 / vCPU / month", volume storage "$0.15 / GB / month", egress "$0.05 / GB"; Hobby "$5 / month", Pro "$20 / month". https://docs.railway.com/pricing/plans

## Ask CopilotKit in writing

1. **Right to resell and host.** May we host OpenBot for several unrelated client organisations as a paid service, and may we install it in a client's own infrastructure, under our brand? Under which agreement? The public terms say "commercial use across your organization" and "internal business purposes".
2. **Who holds the licence.** Is it one licence to us covering every client deployment, or one OpenBot for Business licence per client? May we pass an Intelligence licence or key to a client, given "non-transferable, non-sublicensable"?
3. **"Building agents into your own product".** Does a white-labelled SEO workspace count, so that it "requires a separate Intelligence license"? What does that licence cost and allow?
4. **Billing units in an agency model.** Are a client's staff active users? Are our operators? Is each client an "organization" for the per-user rate and the $5,000 model-spend threshold? What are the per-user tiers below $29?
5. **Licence of the backend after v1.0.3.** What licence covers the `ghcr.io/copilotkit/openbot*` images now that they are compiled and the source has left the repository? Will licensees get the source? May we mirror the images to our own registry and deploy them at clients? May we keep running a fork of the MIT-licensed v1.0.3 server?
6. **Intelligence in production.** May client deployments use managed Intelligence, on which plan, with what data-residency and data-processing terms? If self-hosted, is it one licence per client deployment? Is a licence bound to a machine or cluster?
7. **Multi-tenancy.** The website promises tenants and super-admins; the API spec says one organisation "Until multi-organization lands". Is it available today, in which plan, self-hosted or only on OpenBot Cloud, and what exactly is isolated (database, vault keys, computers, audit, Intelligence project)?
8. **White-label scope.** Confirm we may remove the OpenBot and CopilotKit names, logos, the "OpenBot for Business" entries, the self-hosting banner and product telemetry, and state the attribution you require. May you name our clients as your customers (self-service agreement section 8.5)?
9. **SSO and roles.** Pricing lists "SSO and role-based access control" under Annual. Is using sign-in providers, SCIM and roles on pay as you go permitted?
10. **Trial licence.** How long does the Kubernetes trial licence last, and may a paid client pilot run on it?
11. **OpenBot Cloud.** May we resell a deployment CopilotKit hosts, dedicated per client, and at what price and service level?
12. **Continuity.** What happens to client deployments if the licence lapses or is terminated, what notice applies to price changes, and is the "similar or competitive services" restriction waived for our product?

## Could not verify

- The OpenBot for Business licence and "service terms for the complete offering": not published.
- Per-user tiers below $29 and the graduated per-interaction rates: only in page calculators, not in the page text.
- The trial licence terms: behind an early-access form, which I did not submit.
- Whether multi-tenancy exists in the shipped images: the current server source is not public and nothing was run.
- The Helm guide for the unreleased chart: I read `charts/openbot/README.md` at `v1.0.3`; the chart is no longer in the checkout and I did not download a release asset.
- Behaviour described in `docs/` against the compiled `main` images: the docs still cite source paths that are gone.
- Railway in practice: nothing was deployed. Unchecked in particular is whether a Railway start command replaces the image's `/init` entrypoint, which the one-off sweeps need (`docs/deployment.md:186-194`).
- Dependency licences (`bun.lock`), the bundled Bitnami and embeddings images, and Composio or Arcade terms: not reviewed.
- GitHub issues, discussions and the releases page: the GitHub API for this repository was not available in this session; I used git and the registry instead.
- CopilotKit's privacy policy, data-processing addendum and support terms: not read.
