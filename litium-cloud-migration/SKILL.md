---
name: litium-cloud-migration
description: "Guide Litium partner developers through migrating a customer's Litium 8 site from Litium legacy cloud (Windows/IIS, deployed with Web Deploy or SFTP) to Litium Serverless Cloud: assess the solution and inventory the repo, make the code Linux-ready, build a test environment from legacy backups, replace the Web Deploy pipeline, rehearse with production data, run the cutover, and decommission the legacy site. Tracks progress in a MIGRATION.md file in the customer repo. Triggers: migrate to serverless, legacy cloud, move site, cutover, go-live, WebDeploy, msdeploy, publishsettings, IdentityServer folder, CurrentCulture in scheduled jobs, sql_backup_file, storage_backup_file, Linux-ready, Windows-only package, Fastly or LCC switch, old payment webhook URLs, MIGRATION.md. Delegates all litium-cloud CLI syntax to the litium-cloud-cli skill."
---

# Litium Cloud Migration

## Overview

Partner developers use this skill to move a customer's Litium 8 site from **legacy cloud** (Windows servers with IIS, deployed with Web Deploy or SFTP) to **Serverless Cloud** (Linux containers, deployed as artifacts with the `litium-cloud` CLI). The skill owns the *process*: what to check, in which order, what to ask the customer, when to involve Litium support, and how to run and verify the cutover. It never owns CLI syntax.

Scope:

- **Litium 8.1 or later**; **8.16 or later recommended** (background jobs move to the production worker node; 8.8+ ships the health check endpoints Serverless Cloud probes).
- **Legacy cloud only** as the source. A site on **Litium 7** must be upgraded to Litium 8 first (`litium-developer` covers that). A **self-hosted** site follows the same steps, but backups and domain moves are planned with Litium support rather than requested as legacy-cloud backups.
- MVC Accelerator, React Accelerator (Next.js storefront) and custom storefronts.

Public docs for the process live at https://docs.litium.dev/cloud/serverless/migration/overview; this skill follows those pages and says "confirm with Litium support" wherever they do.

## Delegation

For these areas, delegate entirely to the named skill — do not duplicate its content here:

| Topic | Skill to use |
|-------|-------------|
| Any `litium-cloud` command syntax, flags and output; service principals; secrets; manifest mechanics (`apply`, `marketplace manifest`, `app show -o manifest`); jobs, status and logs; access control | `litium-cloud-cli` (recipes: `new-environment`, `deploy-dotnet`, `deploy-nextjs`, `cicd-service-principal`, `install-litium-platform`, `install-cdn-insights`, `backups`, `restore`, `access-control`, `copy-environment`, `custom-domain`, `app-lifecycle`) |
| General Litium development: accelerator code, data model, APIs, upgrading Litium 7 to 8, back office | `litium-developer` |

When a step needs a command, say **what** to run and **why**, name the `litium-cloud-cli` recipe, and check `litium-cloud <command> --help` before claiming any flag exists.

## Phases

| Phase | Load | Done when |
|-------|------|-----------|
| 1. Assess | `references/assessment.md` | Questionnaire answered, repo inventory done, red flags listed, `MIGRATION.md` created in the customer repo |
| 2. Plan | `references/assessment.md` (red flags), `references/support-requests.md` | Dates, transaction strategy, DNS plan and support requests (backups, access, Fastly) recorded in `MIGRATION.md` |
| 3. Code | `references/code-changes.md` | Solution publishes for Linux, Windows-only code replaced, culture set in jobs, config moved to manifest, first `dotnet` artifact is **Ready** |
| 4. Test environment | `references/environment-setup.md`, `references/manifests.md` | Test environment restored from legacy backups, every app installed, verification list passed, manifests in version control |
| 5. Pipeline | `references/pipeline-migration.md`, `assets/azure-pipelines.yml`, `assets/github-actions.yml` | Pipeline builds a Linux artifact and deploys to test with a service principal, no manual steps |
| 6. Rehearse | `references/rehearsal-and-cutover.md` | Full runbook run in production with fresh production data, every step timed, fixes folded back, rehearsal record in `MIGRATION.md` |
| 7. Cutover | `references/rehearsal-and-cutover.md`, `references/troubleshooting.md` | Domains switched, old webhooks verified, integrations resumed, rollback path known at every stage |
| 8. After | `references/rehearsal-and-cutover.md` (After table), `references/support-requests.md` | Window transactions migrated, toggles removed, legacy environment deleted by support, hand-over done |

## Reference map

Read these files **only** when the user's task requires that area:

| Reference file | Load when the user asks about |
|----------------|-------------------------------|
| `references/assessment.md` | assess a site, inventory, questionnaire, Windows-only packages, `web.config`, `appsettings.*.json`, `Litium:Folder`, scheduled jobs, red flags, how to write `MIGRATION.md` |
| `references/code-changes.md` | make the solution Linux-ready, `System.Drawing`, case-sensitive paths, `CultureInfo.CurrentCulture` in jobs, `Litium.Cloud.Tools.Targets`, probe settings for Litium < 8.8, `license.json`, `.litiumcloudignore`, log files, `/app_storage` paths |
| `references/environment-setup.md` | build the test or production environment, install order (CDN first), restore from `sqlbackup` and `storage` artifacts, force-delete legacy payment apps, SFTP, SMTP relay, storefront, test domain, verification list |
| `references/manifests.md` | which manifests a migrated site needs, `sql_backup_file` and `storage_backup_file`, configurations and `secretRef`, `appRef` to File storage and SMTP relay, pinning payment app versions, keeping manifests identical between environments |
| `references/pipeline-migration.md` | replace Web Deploy or SFTP release, service principal roles, Azure DevOps or GitHub Actions, certificate renewal, where the legacy pipeline steps go |
| `references/rehearsal-and-cutover.md` | rehearsal, runbook, T-1 and T-0 steps, LCC vs non-LCC domain switch, checkout toggle, order prefix, rollback, after go-live, decommissioning |
| `references/support-requests.md` | what to ask support@litium.com for, lead times, request templates, what you get back |
| `references/troubleshooting.md` | app does not start, `console-output`, job failed, file not found, payment app cannot be installed, mail not sent, Insights empty, SFTP cannot connect |
| `assets/MIGRATION.md` | the state file template to copy into the customer repo |
| `assets/azure-pipelines.yml`, `assets/github-actions.yml` | starting points for the replacement pipeline |

## Docs index

Fetch a page as Markdown by appending `.md` to its URL (for example `https://docs.litium.dev/cloud/serverless/migration/go-live.md`).

| Topic | URL |
|-------|-----|
| Migration overview, scope, what changes | https://docs.litium.dev/cloud/serverless/migration/overview |
| Prepare: prerequisites, inventory, support requests, Linux readiness, go-live strategy | https://docs.litium.dev/cloud/serverless/migration/prepare |
| Set up a test environment | https://docs.litium.dev/cloud/serverless/migration/set-up-test-environment |
| Prepare production: pipeline, rehearsal, webhooks, domains, booking support | https://docs.litium.dev/cloud/serverless/migration/prepare-production |
| Go-live runbook and rollback | https://docs.litium.dev/cloud/serverless/migration/go-live |
| After go-live, decommissioning, hand-over | https://docs.litium.dev/cloud/serverless/migration/after-go-live |
| Migration checklist | https://docs.litium.dev/cloud/serverless/migration/checklist |
| Concepts (subscription, environment, app, artifact, manifest, job, secret, appRef) | https://docs.litium.dev/cloud/serverless/concepts |
| Manifest reference | https://docs.litium.dev/cloud/serverless/reference/manifest |
| App actions reference (backup, uninstall-app, add-domain, rebuild-search-indicies, console-output) | https://docs.litium.dev/cloud/serverless/reference/app-actions |
| Roles and permissions | https://docs.litium.dev/cloud/serverless/reference/roles-and-permissions |
| Backups overview, database backup, storage backup, restore | https://docs.litium.dev/cloud/serverless/guides/backups/overview |
| App configuration and secrets | https://docs.litium.dev/cloud/serverless/guides/configure/app-configuration-and-secrets |
| Create a .NET artifact (Targets package, probes, publish for Linux) | https://docs.litium.dev/cloud/serverless/guides/artifacts/create-dotnet-artifact |
| .litiumcloudignore | https://docs.litium.dev/cloud/serverless/guides/artifacts/litium-cloud-ignore |
| Litium platform app (properties, exposed values, actions, worker node) | https://docs.litium.dev/cloud/serverless/apps/public-apps/litium-platform/overview |
| Litium CDN domain app | https://docs.litium.dev/cloud/serverless/apps/public-apps/litium-cdn-domain |
| FAQ and troubleshooting | https://docs.litium.dev/cloud/serverless/faq |
| Legacy deployment (Web Deploy, SFTP, Azure DevOps release) | https://docs.litium.dev/platform/guides/deployment/overview |

## How to use documentation

1. **Live tool output for volatile facts.** Versions, flags, artifact type ids, app type ids and manifest properties change. Use `litium-cloud <command> --help`, `litium-cloud marketplace manifest`, `litium-cloud marketplace list --details` and `litium-cloud artifact artifact-type list` instead of memory. Do not rely on anything the CLI help marks as power-user only.
2. **Bundled references for process knowledge.** The files under `references/` are the migration procedure; load the one for the current phase.
3. **Live docs for detail.** Fetch the page from the Docs index (with `.md` appended) when a reference points to it or when the user asks about something the references do not cover. Some pages are still being written; when a page is a stub, fall back to the bundled reference and say so.
4. **Litium docs MCP server, if configured.** Prefer its search tool over fetching URLs. It is not required.

## Hard rules

- **Destructive actions need an explicit yes.** Never run `app delete`, `environment delete`, `artifact delete`, the `uninstall-app` action, a write `execute-database-script`, or `apply` against a **production** environment without first showing the exact command and getting an explicit yes from the user for that command.
- **Never add `--auto-yes`** unless the user is writing a pipeline and asks for it.
- **Before any production action** run `litium-cloud context show` and `litium-cloud environment show`, and state the target subscription, environment and its production flag in your reply.
- **Never put secrets** in manifests, in `MIGRATION.md`, in pipeline YAML or in chat. Secrets go into environment or subscription secrets and are referenced with `secretRef`.
- **Never author a manifest from memory.** Start from `litium-cloud marketplace manifest` (or `app show -o manifest` for an installed app) and edit.
- **Never set `ASPNETCORE_ENVIRONMENT=Development`**; the app does not start.
- **Never copy the legacy `IdentityServer` folder** into a storage artifact. It holds the legacy app registrations and can break the live legacy site when the apps are force-deleted in the new one.
- **Never change DNS or Fastly, delete the legacy environment, or send email** on the user's behalf. Produce the exact instructions or the support request text (see `references/support-requests.md`) and stop.
- **Do not upload backup artifacts long before go-live** without checking retention: unreferenced artifacts are removed after a retention period (the FAQ states 14 days if never used, 7 days after last use; confirm with Litium support when timing is tight).
- **Check `--help` before claiming a flag exists.** Reference `litium-cloud-cli` for syntax; never invent options.

## Ask vs infer

| Infer from the repo and CLI (do not ask) | Ask the user or customer |
|---|---|
| Litium version (`Litium.*` PackageReference versions), TargetFramework, RuntimeIdentifier | Does the customer have an LCC (Litium Commerce Cloud) agreement? |
| Storefront type (MVC, Next.js, custom) | Does the customer have the App Cloud agreement (needed for File storage, private apps, own MS SQL)? |
| Windows-only packages and code paths | Is the site on Fastly through Litium today? (decides the LCC vs non-LCC domain path) |
| Config keys in `appsettings*.json`, transforms, pipeline variables | Which domains and subdomains move, including redirect domains |
| `Litium:Folder:Local` / `Shared` and other disk paths | Who manages DNS, what the TTLs are, and who has access to change them |
| SMTP keys, scheduled jobs and their culture usage | Which payment and delivery apps are in use, and how long after go-live callbacks for old orders can arrive |
| Existing pipeline YAML, release steps, where `license.json` comes from | SFTP users, their IP addresses and the folders they read and write |
| Subscription and environment ids, installed apps, marketplace versions (`subscription list`, `environment list`, `app list`, `marketplace list --details`) | Go-live date and window, code freeze date, who approves the change freeze |
| Which artifacts exist and their status (`artifact list`, `artifact show`) | Stop sales during the cutover (checkout toggle) or import legacy orders afterwards? New order number prefix? |

## Session flow

1. **Resume first.** If the customer repo contains `MIGRATION.md`, read it before anything else and continue from its *Status* section. If it does not exist and the user is past the first question, run the assessment and create it from `assets/MIGRATION.md`.
2. **One phase at a time.** Load only the reference for the current phase. Update `MIGRATION.md` at the end of every phase (status, ids, decisions, measured durations, open questions) — it is the hand-over document and the record for the next session.
3. **Gates.** Do not start the test environment without a completed inventory. Do not start the cutover without a rehearsal record with measured durations, a confirmed webhook mapping, a written domain plan and a booked support window. If a gate is missing, say which one and go back.
4. **Stop and produce a support request** (from `references/support-requests.md`) whenever a step needs Litium support: subscription access, Fastly access, legacy backups, certificates for domains not in Fastly, old webhook URL mapping, go-live day help, deleting the legacy environment. Support needs at least three working days for scheduled help.
5. **Stop and ask** before every destructive or production action (Hard rules), and whenever a red flag from `references/assessment.md` changes the plan.

## Common mistakes

| Symptom | Root cause | Fix |
|---------|-----------|-----|
| Build or startup fails on Linux with a `PlatformNotSupportedException` or missing native library | A Windows-only package (`System.Drawing.Common`, `Microsoft.Web.Administration`, `System.DirectoryServices*`, `System.Management`, ...) is still referenced | Replace it (see `references/code-changes.md`) and rebuild the artifact |
| Wrong number, date or currency formats in scheduled job output | The OS culture is not set in Serverless Cloud; only web requests get the channel culture | Set `CultureInfo.CurrentCulture` explicitly at the start of every job |
| A legacy payment app cannot be uninstalled, or the legacy site's payment app breaks | The `IdentityServer` folder was copied into the storage artifact | Rebuild the storage artifact without it; never copy that folder |
| A new `sql_backup_file` or `storage_backup_file` in the manifest has no effect | The properties are create-only; `apply` ignores them on an existing app | Uninstall the Litium platform app (and its dependents) and reinstall from the manifest (`litium-cloud-cli` recipe `restore`) |
| Litium < 8.8 app never becomes ready after install or deploy | No built-in health endpoints; the probes never succeed | Set the probe paths to `"none"` or implement the endpoints, see `references/code-changes.md` |
| Values from `appsettings.Staging.json` (or any other environment file) are missing | Only `appsettings.json` and `appsettings.production.json` are loaded | Move the values to configurations and secrets in the manifest |
| `File not found` for a file that is in the artifact | Linux paths are case-sensitive | Match folder and file name case exactly in code and configuration |
| Payment callbacks for orders placed before go-live return 404 | The old webhook URLs were not mapped in the new Fastly service | Send the old URL list to support before go-live (`references/support-requests.md`) and verify on T-0 |
| The storage artifact is gone on go-live day | Uploaded too early and never referenced; removed by retention | Upload on T-1, verify **Ready**, note the id; download a copy you must keep |
| Mail is not sent, or lands in spam | No SMTP relay app, SMTP keys still point at the legacy server, or SPF not updated for the new sender | Install the SMTP relay app, reference its exposed values, and have the customer update SPF |
| Integration files are missing after go-live | Code still reads a legacy disk path; no File storage mount | Mount a File storage folder with `type: storage` and read from `/app_storage/<name>` |
| Litium Insights shows nothing for the app | The app never started, so nothing reached Insights | Run the `console-output` action and read the job log (`references/troubleshooting.md`) |
| `apply` reports *unchanged* or ignores a directory | A bare directory was passed to `--file`, or a typo in a manifest key (unknown keys are ignored) | Use `<dir>/*.yaml`; check the result with `app show` |
