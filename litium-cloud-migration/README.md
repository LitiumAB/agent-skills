# litium-cloud-migration

A process skill for Litium partner developers migrating a customer's Litium 8 site from Litium legacy cloud
(Windows, IIS, Web Deploy) to Litium Serverless Cloud (Linux containers, artifacts, manifests). It owns the
*procedure* — what to check, in which order, what to ask the customer, when to involve Litium support, and how
to run and verify the cutover — and tracks the whole migration in a `MIGRATION.md` file in the customer's
repository, so the next session, or the next person, picks up exactly where the last one stopped.

## Install

```bash
npx skills add https://github.com/LitiumAB/agent-skills --skill litium-cloud-migration
```

## What it does

- Assesses the legacy solution: questionnaire, repo inventory, and the red flags that change the plan
- Makes the solution Linux-ready: Windows-only packages, case-sensitive paths, culture in background jobs,
  configuration out of `appsettings` files, the license and the first `dotnet` artifact
- Builds a test environment from the legacy database and files backups, in install order, with every app the
  site needs — CDN, Insights, platform, storefront, payment and delivery, File storage, sFTP, SMTP relay,
  domains
- Replaces the Azure DevOps Web Deploy release with an artifact pipeline, with ready-to-adapt Azure DevOps and
  GitHub Actions templates, service principal roles and certificate renewal
- Writes the app manifests from the downloaded templates and keeps test and production identical
- Runs the rehearsal and the cutover from a timed runbook, with go / no-go points and a rollback path at every
  stage
- Drafts the support requests Litium has to handle — backups, Fastly access, certificates, go-live help,
  decommissioning — and stops for you to send them
- Troubleshoots the failures that actually happen during a migration, from symptom to verified fix

## Works together with

- **litium-cloud-cli** — every `litium-cloud` command, flag and output. This skill names the recipe to follow
  and never duplicates CLI syntax.
- **litium-developer** — accelerator code, data model, APIs, back office, and upgrading Litium 7 to 8 before a
  migration can start.

## No configuration required
