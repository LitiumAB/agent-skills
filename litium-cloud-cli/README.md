# litium-cloud-cli

A skill for working with Litium Serverless Cloud through the `litium-cloud` CLI — creating
environments, installing and updating apps from YAML manifests, packaging and deploying artifacts,
managing secrets and access control, taking and restoring backups, adding custom domains, and
automating all of it from a CI/CD pipeline.

## Install

```bash
npx skills add https://github.com/LitiumAB/agent-skills --skill litium-cloud-cli
```

## What it does

- Creates and configures environments, and sets a per-folder context so commands target the right one
- Installs and upgrades apps from marketplace manifests with `litium-cloud apply`
- Packages .NET, Next.js, Node.js and Nuxt.js code as artifacts and deploys them with rolling updates
- Sets up service principals and the four minimum roles a deployment pipeline needs
- Manages subscription and environment secrets, and references them from manifests instead of literals
- Grants, inspects and removes access, including groups and inheritance on production environments
- Takes database and storage backups, downloads them, and restores an environment from them
- Adds, replaces and removes custom domains through Litium CDN and the platform domain actions
- Restarts, pauses, resumes, re-plans and uninstalls apps, and follows every job to its real result
- Keeps to the published Litium documentation and verifies flags against the installed CLI's `--help`

## Works together with

- **litium-cloud-migration** — the process for moving a site from Litium legacy cloud to Serverless Cloud; it
  names the recipes in this skill for every command it needs.
- **litium-developer** — application code, accelerators, data model, APIs and back office.

## No configuration required
