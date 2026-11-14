# SEO in a Box

A product that gives a Client's website a team of AI Bots that do SEO work, directed by an Operator who answers for anything that takes effect outside the Bots' own computers.

## Language

### People and Instances

**Client**:
The business whose website the SEO work is for, and who pays for it.
_Avoid_: Customer, tenant, account

**Operator**:
The person who directs the Bots and answers their Approvals on a Client's behalf.
_Avoid_: User, admin, owner

**Instance**:
One running copy of the product, serving exactly one Client.
_Avoid_: Deployment, tenant, workspace

### Sites

**Site**:
A website the Bots work on. An Instance can hold several.
_Avoid_: Project, property, domain

**Git site**:
A Site whose content and code live in a git repository.

**CMS site**:
A Site whose content lives in a content management system.

### Work

**Bot**:
An AI worker with one SEO role, a computer of its own, and a record of everything it does.
_Avoid_: Agent, assistant, coworker

**SEO skill**:
A named, repeatable way of doing one SEO task, which a Bot follows.
_Avoid_: Skill on its own (it also means the agent skills used to build this repo), workflow, playbook

**Conversation**:
The exchange in which an Operator gives a Bot work and the Bot reports back.
_Avoid_: Thread, channel

**Routine**:
Work a Bot starts without being asked.
_Avoid_: Job, cron, automation

**Deliverable**:
Something a Bot produced for a Site that outlives the Conversation it came from.
_Avoid_: Output, artifact, asset

**Ship step**:
The action that makes a Bot's work take effect outside its own computer, on the Site or with another person.
_Avoid_: Publish, deploy, push

**Approval**:
An Operator's yes or no to a Ship step, given before it happens.
_Avoid_: Review, sign-off, permission

**Activity**:
The record of what a Bot did, what it was refused, and what it was approved to do.
_Avoid_: Audit log, history, logs

### The interface

**Workbench**:
The Operator's single screen for directing Bots and answering Approvals.
_Avoid_: Dashboard, app, console
