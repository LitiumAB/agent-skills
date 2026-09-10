---
name: litium-cloud-cli
description: "Manage Litium Serverless Cloud with the litium-cloud CLI: environments, apps, artifacts, manifests, secrets, access control and pipeline deployments. Use for deploy to Litium Cloud or Serverless Cloud, create environment, install Litium platform, Litium CDN, Litium Insights, storefront, payment or delivery apps, create or upload an artifact, write or apply a YAML manifest, app deploy, app action, service principal, CI/CD pipeline deployment, access control, roles, groups, subscription and environment secrets, database and storage backups, restore, copy environment, custom domain, job status and logs, and the Litium Cloud Portal. Triggers: litium-cloud, cloud CLI, deploy to Litium Cloud, create environment, install Litium platform, artifact, manifest, apply, service principal, pipeline deployment, access control, group, secret, backup, restore, copy environment, custom domain, app action, job status, Portal."
---

# Litium Cloud CLI

## Overview

Litium Serverless Cloud hosts Litium platform, storefronts, payment and delivery apps and your own
code as **apps** inside **environments** inside a **subscription**. The `litium-cloud` CLI is the
front end for all of it, and the only way to upload artifacts.

Use this skill for every Litium Cloud task: creating environments, installing and updating apps from
YAML manifests, packaging and deploying code artifacts, secrets, access control, backups, custom
domains and CI/CD pipelines. Load the reference files below on demand — do not guess a command.

## Prerequisites

| Requirement | How |
|---|---|
| .NET 8 SDK | https://dotnet.microsoft.com/download/dotnet/8.0 (needed even with newer SDKs installed) |
| Litium NuGet feed | `dotnet nuget add source https://nuget.litium.com/nuget/ -n Litium -u <docs-username> -p <docs-password>` (add `--store-password-in-clear-text` on macOS/Linux) |
| Install or update the CLI | `dotnet tool update -g litium.cloud.cli --no-cache` |
| Verify | `litium-cloud --version` (prints e.g. `2.10.1`) |
| Sign in | `litium-cloud auth login` — opens a browser for the **Litium Account** (not the docs account) |

The tool lands in `~/.dotnet/tools` (macOS/Linux) or `%USERPROFILE%\.dotnet\tools` (Windows); add it
to `PATH` if the shell cannot find `litium-cloud`.

## Dynamic help first

The installed CLI is the authority on **flags**; the reference files are the authority on **process**
and on what a command means. Before using any option you are not certain of:

```bash
litium-cloud <group> --help
litium-cloud <group> <command> --help
```

Three known mistakes in the 2.10.x help text — do not copy them:

| Help text says | Reality |
|---|---|
| `artifact-type list` | The real path is `litium-cloud artifact artifact-type list` (nested under `artifact`) |
| `app search`, `app search --detail` | No such command. Use `litium-cloud marketplace list --details` |
| `service-principal --help` omits `renew` | `litium-cloud service-principal renew` exists and works; `update` is a deprecated alias |

## Key concepts

Read `references/concepts.md` for the detail. In brief:

- **Hierarchy**: subscription → environment → app. Artifacts and subscription secrets live on the subscription.
- **App type vs installed app id**: `litium-platform` is the type; `litium` is the id you pass to `--app`.
- **Apps are installed with `apply`**, never with an `app install` command. There is no `app create`.
- **Manifests** are YAML (`kind` / `resource.id` / `spec`). Always start from `marketplace manifest`.
- **Artifacts** are immutable uploads owned by a subscription: `dotnet`, `nextjs`, `nodejs`, `nuxtjs`, `sqlbackup`, `storage`.
- **Code artifacts** are deployed any time with `app deploy`; **backup artifacts** apply only at install.
- **Secrets** live on a subscription or an environment; the environment value wins when both exist.
- **Jobs**: nearly every mutating command returns a job id and exits 0 immediately. The job may still fail.
- **Access control**: role + principal + scope, inherited downwards; `--convert` on `disable-inheritance`.
- **Portal** (https://portal.litium.cloud) does everything except uploading artifacts.

## Reference map

| Reference file | Load when the task involves |
|---|---|
| `references/concepts.md` | Hierarchy, app types and versions, plans, artifact types, manifest grammar, configurations and `valueFrom`, context file, auth, roles and inheritance, job states, artifact retention, Portal vs CLI |
| `references/commands.md` | Exact syntax of any `litium-cloud` command group, required and notable options, which commands start jobs, global options, exit codes, environment variables, CI behaviour |
| `references/workflows.md` | A complete end-to-end task — the twelve recipes listed under Workflow routing |

## Workflow routing

Load `references/workflows.md` and follow the named recipe.

| User intent | Recipe |
|---|---|
| Set up a brand-new environment from scratch | `new-environment` |
| Build and deploy .NET code (Litium platform, MVC accelerator, .NET web app) | `deploy-dotnet` |
| Build and deploy a Next.js storefront | `deploy-nextjs` |
| Automate deployments from Azure DevOps, GitHub Actions or any pipeline | `cicd-service-principal` |
| Install Litium platform for the first time, with or without backups | `install-litium-platform` |
| Install Litium CDN and Litium Insights | `install-cdn-insights` |
| Take a database or storage backup, or download one | `backups` |
| Restore a database and storage backup into an environment | `restore` |
| Grant, remove or inspect access; groups; inheritance | `access-control` |
| Replicate one environment into another | `copy-environment` |
| Put a customer domain in front of an app | `custom-domain` |
| Restart, pause, resume, re-plan, uninstall an app; delete an environment | `app-lifecycle` |

## Docs index

Public docs at `https://docs.litium.dev/<path>`. **Append `.md` to any URL** to fetch the plain
Markdown (for example `https://docs.litium.dev/cloud/serverless/cli/app.md`). The docs also expose an
optional MCP server at `https://docs.litium.dev/mcp` for live search — use it when it is configured.

| Topic | Path |
|---|---|
| Concepts and vocabulary | `/cloud/serverless/concepts` |
| CLI overview, global options, exit codes | `/cloud/serverless/cli/overview` |
| Command references | `/cloud/serverless/cli/{app,apply,artifact,auth,context,environment,group,location,marketplace,query,role,service-principal,status,subscription}` |
| Manifest reference | `/cloud/serverless/reference/manifest` |
| App actions per app type | `/cloud/serverless/reference/app-actions` |
| Roles and permissions | `/cloud/serverless/reference/roles-and-permissions` |
| Apps: public vs private, install procedure | `/cloud/serverless/apps/overview` |
| Artifacts: types, ignore files, retention | `/cloud/serverless/guides/artifacts/overview` |
| Create a .NET / Next.js artifact | `/cloud/serverless/guides/artifacts/create-dotnet-artifact`, `…/create-nextjs-artifact` |
| Deploy an artifact, roll back | `/cloud/serverless/guides/artifacts/deploy-artifact` |
| Automated deployments (+ Azure DevOps, GitHub Actions) | `/cloud/serverless/guides/automated-deployments/overview` |
| App configuration and secrets | `/cloud/serverless/guides/configure/app-configuration-and-secrets` |
| Custom domains | `/cloud/serverless/guides/configure/custom-domains` |
| Backups and restore | `/cloud/serverless/guides/backups/overview`, `…/database-backup`, `…/storage-backup`, `…/restore-database-and-storage` |
| Access control, groups, service principals | `/cloud/serverless/guides/access/overview`, `…/access-control`, `…/groups`, `…/service-principals` |
| Jobs, app lifecycle, copy environment, SQL scripts, console output | `/cloud/serverless/guides/operate/jobs-status-and-logs`, `…/manage-app-lifecycle`, `…/copy-environment`, `…/execute-sql-script`, `…/console-output` |
| Get started, step by step | `/cloud/serverless/get-started/overview` |
| FAQ and troubleshooting | `/cloud/serverless/faq` |
| Portal | `/cloud/serverless/portal/overview` |

## Rules

1. **Confirm the target before any mutating command.** Run `litium-cloud context show` (and
   `litium-cloud auth show`) and state out loud which subscription and environment the command will
   hit, and **whether that environment is a production environment** — check with
   `litium-cloud environment show`, which prints `Production: Yes|No`.
2. **Never add `--auto-yes` to a destructive command** (`app delete`, `app pause`) unless you are
   writing a pipeline script the user asked for. It suppresses the safety prompt.
3. **`environment delete` has no confirmation prompt at all.** It starts immediately. Read the ids
   back to the user before running it.
4. **Never put a secret in a manifest.** Create it with `subscription secret create` or
   `environment secret create` and reference it with `valueFrom.secretRef`. Prefer `--text-value` /
   `--binary-value` over `-v` so the value never lands in shell history or a build log.
5. **Start every manifest from `litium-cloud marketplace manifest --app <app-type> -f <file>.yaml`.**
   That template is the authority on the properties and configurations an app version accepts. Never
   hand-write a manifest from memory, and never invent a top-level key such as `apiVersion`.
6. **Follow every job.** Mutating commands print a job id and exit 0 as soon as the job is *queued*.
   Run `litium-cloud status logs --job <job-id> --follow`, then `litium-cloud status show --job <job-id>`
   and check that no entry under **Details** has status *failed* — a parent job says `completed`
   even when a child failed.
7. **Use `-o json` for anything scripted or parsed.** Never parse the default table output.
8. **The Portal covers everything except uploading artifacts.** Offer it as the alternative when the
   user prefers a browser: https://portal.litium.cloud.
9. Prefer setting a context per project folder over repeating `--subscription` and `--environment`.
10. Do not use commands or options that the installed `--help` does not list, other than
    `service-principal renew` (see Dynamic help first).

## Common mistakes

| Symptom | Cause | Fix |
|---|---|---|
| `litium-cloud artifact-type list` → command not found | `artifact-type` is nested under `artifact` | `litium-cloud artifact artifact-type list` |
| `app search` → command not found | The command does not exist; the help text is wrong | `litium-cloud marketplace list --details` |
| `apply` skips everything / does nothing for a folder | `-f` was given a bare directory | Pass a file or a glob: `-f manifests/*.yaml` |
| A manifest with `action: delete` and `kind: app` fails | `action: delete` is not implemented for apps | Use `litium-cloud app delete --app <app-id>` |
| Command succeeded but nothing changed | The command only queued a job, and the job failed | `status show --job <id>` and read **Details** for a *failed* entry |
| `Artifact is type '<x>' but application require type '<y>'` | Wrong artifact type for the app | `dotnet` for platform/.NET web, `nextjs` for the storefront; check `artifact list` |
| The artifact id is gone | Unreferenced artifacts are deleted: 14 days if never used, 7 days after last use | Rebuild from source; keep releases in your pipeline, not in the cloud |
| A restored backup did not take effect | `sql_backup_file` and `storage_backup_file` are create-only | Uninstall and reinstall the app with the artifact ids in the manifest |
| The app does not see a new secret or configuration value | Apps read configuration at startup | `litium-cloud app restart --app <app-id> --wait` |
| `Aborting destructive operation. Use --auto-yes …` | Non-interactive session (CI detected) | Add `--auto-yes` only in a deliberate pipeline script |
| `Unauthenticated. Interactive login not possible.` in a pipeline | No browser in CI | `auth login --service-principal --username <id> --certificate <path>` |
| Pipeline signs in but every command is denied | A service principal inherits nothing from its creator | Grant it roles explicitly — see the `cicd-service-principal` recipe |
| `Could not connect to server.` after moving directory | Relative certificate path in `.litium-cloud.config` | Use an absolute certificate path; re-run with `-d` to see the real error |
| A deploy is rejected: `App is currently paused.` | The app is paused | `litium-cloud app resume --app <app-id>`, wait, then deploy |
