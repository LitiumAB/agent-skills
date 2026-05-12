---
name: litium-cloud-cli
description: "Skill for managing Litium Serverless Cloud environments using the `litium-cloud` CLI. Use this skill whenever the user wants to: create or manage cloud environments, deploy code to Litium Cloud (MVC Accelerator, Next.js storefront), install Litium Platform, CDN, Insights or payment/delivery apps, create or download artifacts, set up CI/CD pipelines with service principals, manage access control and user groups, copy environments, create database or storage backups, apply YAML manifests, or run any `litium-cloud` CLI command. Trigger phrases include: 'deploy to cloud', 'create environment', 'litium-cloud', 'cloud CLI', 'cloud deployment', 'install Litium platform', 'service principal', 'YAML manifest apply', 'artifact', 'cloud subscription', 'copy environment', 'cloud backup', 'automate deployment'."
---

# Litium Cloud CLI

Skill for managing Litium Serverless Cloud using the `litium-cloud` CLI tool. Covers all common partner-developer tasks: environment setup, app installation, artifact deployment, CI/CD automation, access control, and environment copying.

## Pre-requisite

The `litium-cloud` CLI must be installed on the local machine:

```bash
dotnet tool update -g litium.cloud.cli --no-cache
```

Verify installation: `litium-cloud --version`

## Dynamic help

Because the CLI is installed locally, always check the latest flags by running:

```bash
litium-cloud --help
litium-cloud <command> --help
litium-cloud <command> <subcommand> --help
```

Use the reference files below for baseline knowledge; use `--help` for up-to-date parameter details.

## Key concepts

Read `references/concepts.md` when the user needs to understand:
- The subscription → environment → app resource hierarchy
- What an artifact is and the supported types (dotnet, nextjs, nodejs, nuxtjs, sql-backup, storage-backup)
- How YAML manifests work with `litium-cloud apply`
- How context works (`litium-cloud context set`) and why it avoids repeating `--subscription`/`--environment`
- Authentication: interactive login vs. service principal
- Access control: roles, groups, ACL inheritance

## Reference map

Load the relevant reference when the user's task falls in that area:

| Reference file | Load when the user asks about |
|---|---|
| `references/concepts.md` | Subscription/environment/app hierarchy, artifacts, manifests, authentication, access control model |
| `references/commands.md` | Any specific command syntax, subcommands, available options, or examples for the 14 standard commands |
| `references/workflows.md` | Step-by-step task: new environment setup, deploy .NET/Next.js artifact, install platform/CDN/Insights, CI/CD service principals, backups, access management, copy environment |

## Command groups (standard)

Run `litium-cloud --help` to see the live list. The 14 standard commands available to all partner developers are:

| Command | Purpose |
|---|---|
| `apply` | Apply YAML manifest to install/configure apps |
| `app` | List, deploy, pause, restart, delete apps; run actions |
| `artifact` | Create, list, download, delete deployment artifacts |
| `auth` | Login, logout, show current user, manage service principals login |
| `context` | Set default subscription/environment to avoid repeating flags |
| `environment` | Create, list, update, delete environments |
| `group` | Create/manage user groups for access control |
| `location` | List available deployment regions |
| `marketplace` | Browse available apps and download their YAML manifests |
| `role` | List available roles |
| `service-principal` | Create/manage service accounts for CI/CD |
| `status` | View job status and logs |
| `subscription` | List and view subscriptions |

## Workflow routing

For end-to-end tasks, load `references/workflows.md` and follow the matching recipe:

| User intent | Workflow |
|---|---|
| New Litium Cloud setup from scratch | `new-environment` |
| Deploy updated .NET code (MVC Accelerator) | `deploy-dotnet` |
| Deploy updated Next.js storefront | `deploy-nextjs` |
| Set up automated deployments / pipelines | `cicd-service-principal` |
| First-time Litium Platform install | `install-litium-platform` |
| Install CDN and Insights | `install-cdn-insights` |
| Create or restore database/storage backup | `backups` |
| Grant or revoke user/group access | `access-control` |
| Clone test environment to production | `copy-environment` |

## General guidelines

- Always run `litium-cloud auth show` first to confirm the user is logged in.
- If the user hasn't set context, suggest `litium-cloud context set --subscription <id> --environment <id>` — it makes subsequent commands shorter.
- After running `apply` or `app deploy`, note the job ID and offer to check status with `litium-cloud status show --job <id>`.
- For production environments, remind the user to include `--production` when creating the environment.
- Artifacts support rolling deployments (zero downtime). Inform the user after `app deploy` completes.
- The `litium-cloud marketplace manifest` command is the canonical way to get install YAML for any public app (litium-platform, litium-cdn, litium-insights, litium-storefront, etc.).
- The `--output json` (or `-o json`) flag is useful for scripting. Use it when the user needs to parse output.
- Always check `references/commands.md` before guessing a flag name — then confirm the exact spelling with `--help`.

## Documentation

Official cloud docs: https://docs.litium.dev/cloud/serverless/overview
Getting started guide: https://docs.litium.dev/cloud/serverless/get-started/getting-started-with-cloud-cli
