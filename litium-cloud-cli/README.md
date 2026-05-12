# litium-cloud-cli

A skill for managing Litium Serverless Cloud environments using the `litium-cloud` CLI. Covers environment creation, app installation, artifact deployment, CI/CD automation with service principals, access control, and environment copying.

## Install

```bash
npx skills add https://github.com/LitiumAB/agent-skills --skill litium-cloud-cli
```

## Pre-requisite

The `litium-cloud` CLI must be installed on the developer's machine:

```bash
dotnet tool update -g litium.cloud.cli --no-cache
```

## What it does

- Creates and manages cloud environments (dev, staging, production)
- Installs public apps: Litium Platform, CDN, Insights, Storefront, payment and delivery apps
- Builds and uploads .NET (MVC Accelerator) and Next.js artifacts
- Deploys artifacts to running apps with zero-downtime rolling deployments
- Sets up CI/CD pipelines using service principals with certificate authentication
- Manages access control: roles, user groups, ACL inheritance
- Creates and restores database and storage backups
- Copies environments (e.g. replicating test to production)
- Applies YAML manifests declaratively with `litium-cloud apply`
- Monitors job status and streams logs

## Sample usage

Once installed, the skill is automatically available to your AI agent:

```
Create a new Litium Cloud environment called "staging" in my subscription.
```

```
Build and deploy the MVC Accelerator to Litium Cloud.
```

```
Set up a CI/CD service principal so GitHub Actions can deploy automatically.
```

```
Copy the production environment to a new test environment.
```

```
Create a database backup and download it.
```

```
Grant the QA team reader access to the production environment.
```
