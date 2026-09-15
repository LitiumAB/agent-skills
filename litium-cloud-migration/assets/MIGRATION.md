# Migration to Litium Serverless Cloud

State file for this site's migration from Litium legacy cloud to Serverless Cloud. It is the hand-over
document and the record the next session starts from.

> **How to use it.** Keep it in the root of the solution repository and commit it with the code. The agent
> running the `litium-cloud-migration` skill reads it first and updates it at the end of every phase: status,
> ids, decisions, measured durations, open questions.
>
> **Never write secrets here.** No passwords, connection strings, API keys, certificate contents or SFTP
> passwords — not even temporarily. Subscription, environment, app, artifact, job and secret **ids** are fine;
> secret **values** are not.

---

## Status

| | |
|---|---|
| Current phase | <1 Assess / 2 Plan / 3 Code / 4 Test environment / 5 Pipeline / 6 Rehearse / 7 Cutover / 8 After> |
| Last updated | <date> by <name> |
| Customer contact | <name, role> |
| Partner owner | <name> |
| Subscription id | <id> |
| Test environment id | <id> |
| Production environment id | <id> (production flag: <yes/no>) |
| Go-live date and window | <date, start–end, time zone> |
| Next action | <one line> |

### Phase checklist

**Prepare**

- [ ] Litium 8.1 or later confirmed (8.16+ recommended), or the upgrade planned first
- [ ] LCC agreement confirmed; App Cloud agreement confirmed if private apps are needed
- [ ] Subscription access granted and the CLI signed in
- [ ] Legacy solution inventoried (see *Inventory*)
- [ ] Database backup and files backup (without `IdentityServer`) requested from support
- [ ] Fastly access requested, if the partner will manage domains
- [ ] `Litium.Cloud.Tools.Targets` added and the solution publishes for Linux
- [ ] Windows-only dependencies replaced and case-sensitive paths fixed
- [ ] `CurrentCulture` set explicitly in background jobs
- [ ] Environment-specific settings and secrets moved to configurations and secrets
- [ ] Non-media file paths moved to a File storage mount
- [ ] Dependencies on log files on disk removed
- [ ] `license.json` in the artifact; `.litiumcloudignore` added if needed
- [ ] First `dotnet` artifact built and **Ready**
- [ ] Code freeze date, go-live date and window set
- [ ] Transaction strategy decided (checkout toggle, later import, order prefix)
- [ ] DNS TTL plan agreed, if the site is not on Fastly today

**Test environment**

- [ ] Test environment created (no production flag) and context set
- [ ] Litium CDN installed
- [ ] Litium Insights installed and receiving logs
- [ ] Legacy database uploaded as a `sqlbackup` artifact
- [ ] Legacy files uploaded as a `storage` artifact, without `IdentityServer`
- [ ] Litium platform installed with `artifact`, `sql_backup_file`, `storage_backup_file`
- [ ] Legacy payment and delivery apps force-deleted; new ones installed and configured
- [ ] File storage and one SFTP app per user installed, if SFTP is used
- [ ] SMTP relay installed, if the site sends mail
- [ ] Storefront app installed, if the site is headless
- [ ] Other private apps installed
- [ ] Test custom domain added
- [ ] Verification list passed with the customer
- [ ] Every manifest in version control under `deploy/serverless/test/`

**Pipeline**

- [ ] Service principal created with the minimum roles; certificate stored as a pipeline secret
- [ ] Pipeline publishes for Linux, creates a `dotnet` artifact and deploys to test with no manual step
- [ ] Legacy release pipeline, publish profiles and config transforms removed from the repo
- [ ] Certificate expiry date recorded (see *Hand-over*)

**Prepare production**

- [ ] Production environment created with the production flag
- [ ] Access restricted; the service principal has only the deployment roles
- [ ] Litium CDN and Litium Insights installed from the test manifests
- [ ] Production manifests checked in under `deploy/serverless/production/`; every secret created
- [ ] Fresh production backups obtained and the full runbook rehearsed and timed
- [ ] Test orders, payment callbacks, integrations and jobs verified with production data
- [ ] Every inbound webhook URL inventoried, old and new
- [ ] Old payment webhook URLs sent to support and the mapping confirmed
- [ ] Domain list complete; LCC or non-LCC path decided; owner per domain named
- [ ] Certificates requested for domains not already in Fastly
- [ ] DNS TTL lowered to 300 s at least a day before go-live (non-LCC)
- [ ] Support booked for go-live day, at least three working days ahead
- [ ] Change freeze communicated to the customer's administrators in writing

**Go live**

- [ ] T-1: freeze and bookings confirmed
- [ ] T-1: dependent apps and the Litium platform app uninstalled
- [ ] T-1: files backup taken and uploaded as a `storage` artifact
- [ ] T-0: legacy integrations paused, checkout toggle on, database freeze announced
- [ ] T-0: final database backup taken and uploaded as a `sqlbackup` artifact
- [ ] T-0: platform manifest updated with the three artifact ids and committed
- [ ] T-0: platform manifest applied and the job finished successfully
- [ ] T-0: legacy apps force-deleted; payment, delivery, storefront and other apps reinstalled
- [ ] T-0: search indexes rebuilt
- [ ] T-0: smoke test passed on the system domain, including a test order
- [ ] T-0: domains switched and the checkout toggle turned off
- [ ] T-0: old and new webhook URLs verified against a real order
- [ ] T-0: integrations resumed in the new site
- [ ] Rollback path known and agreed at every stage

**After go-live**

- [ ] New site monitored in Litium Insights
- [ ] Legacy site monitored for stray traffic; each source fixed
- [ ] Orders and other transactions from the switch window migrated
- [ ] Every scheduled job and integration confirmed to have run
- [ ] Temporary toggles removed and deployed through the pipeline
- [ ] Final legacy backups stored; legacy environment deletion requested from support
- [ ] DNS TTL restored (non-LCC)
- [ ] Hand-over complete (see *Hand-over*)

---

## Inventory

| Item | Finding | Evidence (file or command) |
|---|---|---|
| Litium version (code / database) | | |
| Target framework | | |
| Storefront type | | |
| Integrations and scheduled jobs | | |
| SFTP users, folders, IP addresses | | |
| Email sending (SMTP keys, sender) | | |
| Payment and delivery apps + versions | | |
| Inbound webhook URLs | | |
| Domains and subdomains, DNS owner, on Fastly today? | | |
| File storage paths (`Litium:Folder:*` and others) | | |
| Windows-only dependencies | | |
| Configuration files, transforms, pipeline variables | | |
| License (`license.json`) source | | |
| Deployment pipeline today | | |
| Media and database size | | |

---

## Red flags

| Finding | Consequence | Handled how |
|---|---|---|
| | | |

---

## Decisions

| # | Date | Decision | Made by | Rationale |
|---|------|----------|---------|-----------|
| 1 | | Go-live date and window: | | |
| 2 | | Code freeze date: | | |
| 3 | | Transactions during the switch: <freeze / continue and import> | | |
| 4 | | New order number prefix: | | |
| 5 | | Domain path: <LCC Fastly move / DNS switch> | | |
| 6 | | Who moves the domains: | | |
| 7 | | SMTP: <customer's own service / Litium relay> | | |

---

## Support requests

| # | Type | Request | Sent | Due | Status | Received (ids, confirmations) |
|---|------|---------|------|-----|--------|-------------------------------|
| 1 | Subscription access | | | | | |
| 2 | Fastly access / test domain | | | | | |
| 3 | Legacy backups | | | | | |
| 4 | Go-live day help | | | | | |
| 5 | Certificates (non-LCC) | | | | | |
| 6 | Delete legacy environment | | | | | |

Scheduled help needs at least three working days of lead time.

---

## Open questions

| # | Question | Owner | Asked | Answer |
|---|----------|-------|-------|--------|
| 1 | | | | |

Everything marked "confirm with Litium support" or "ask the customer" belongs here until it is answered.

---

## Rehearsal record

Date: <date>. Artifacts used: code `<id>`, database `<id>`, storage `<id>`.

| Step | Duration | Notes |
|---|---|---|
| Files backup taken (legacy side) | | |
| Storage artifact upload → Ready | | |
| Uninstall dependents + platform | | |
| Database backup taken (legacy side) | | |
| Database artifact upload → Ready | | |
| Platform apply → Running | | |
| Force-delete legacy apps | | |
| Dependent apps applied and registered | | |
| Search index rebuild | | |
| Smoke test | | |
| **Total from freeze to verified** | | |

Fixes made (all folded into code or manifests, none applied by hand in the environment):

- 

Manual steps found (each becomes a runbook line):

- 

Test order results per payment and delivery method:

- 

---

## Cutover plan

Support contact during the window: <name, channel, phone>. Go / no-go decider: <name>.

| Time | Step | Owner | Expected duration | Verification | Rollback |
|---|---|---|---|---|---|
| T-1 | | | | | |
| T-0 | | | | | |

Domain list:

| Domain | Target app | Certificate confirmed | Who moves it | Order |
|---|---|---|---|---|
| | | | | |

Webhook mapping (old URL → new target, who updates the external system):

| Old URL | New target | Owner | Verified on T-0 |
|---|---|---|---|
| | | | |

---

## Cutover log

Filled in live on the day. Actual times, job ids, artifact ids, deviations, who did what.

| Time | Step | Actual duration | Job / artifact id | Outcome | Who |
|---|---|---|---|---|---|
| | | | | | |

---

## After

- Monitoring notes (errors, order volume compared with the same weekday before):
- Stray traffic to the legacy site and how each source was fixed:
- Window transactions reconciled (count confirmed by the customer):
- Scheduled jobs and integrations confirmed running:
- Temporary toggles removed on:
- Final legacy backups stored at (location, who has access):
- Legacy environment deletion confirmed by support on:
- DNS TTL restored on:

---

## Hand-over

- Manifests: `deploy/serverless/<env>/` in this repository, one file per app.
- Pipeline: <location>, service principal `<id>`, certificate expires `<date>`, secret stored in <where>.
- Secrets: ids and what each is for, with the owner who can rotate it (values are not recorded here).

| Secret id | Purpose | Owner |
|---|---|---|
| | | |

- Access: who has which role on the subscription, the environments and the apps.
- SFTP: host, user and folders per external system, and the IP allow list (passwords are read with the CLI).
- Webhook inventory: the table under *Cutover plan*, kept current.
- Measured durations: the tables under *Rehearsal record* and *Cutover log*.
- Runbooks the customer needs: backups and restore, deploying a release (database backup first, previous artifact
  id noted for rollback), adding a domain, reading logs in Litium Insights, opening a support case.
