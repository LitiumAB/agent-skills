# Litium Agent Skills

A collection of AI agent skills for building on the Litium platform. Works with GitHub Copilot, Cursor, Windsurf, and any other agent that supports the skills standard.

## Available Skills

| Skill | Description |
|-------|-------------|
| [litium-developer](./litium-developer/README.md) | Comprehensive development skill for Litium partner developers — covers React Accelerator (Next.js), MVC Accelerator (.NET), backoffice UI extensions (React, Vue, Angular, Vanilla JS via `@litiumab/platform-extension-sdk`), data modelling, APIs, local setup, and troubleshooting. |
| [litium-cloud-cli](./litium-cloud-cli/README.md) | Manage Litium Serverless Cloud with the `litium-cloud` CLI — environments, apps and manifests, artifacts and deployments, secrets, access control, backups and restore, custom domains, and CI/CD pipelines with service principals. |
| [litium-cloud-migration](./litium-cloud-migration/README.md) | Guide a migration of an existing Litium 8 site from Litium legacy cloud to Serverless Cloud — assessment, code changes for Linux, environment setup, pipeline conversion, rehearsal, go-live runbook and support requests. Works together with `litium-cloud-cli`. |

## Installing a Skill

Skills are installed with the [`skills` CLI](https://www.npmjs.com/package/skills):

```bash
npx skills add https://github.com/LitiumAB/agent-skills --skill <skill-name>
```

### Install `litium-developer`

```bash
npx skills add https://github.com/LitiumAB/agent-skills --skill litium-developer
```

This copies the skill files into your project's `.agents/skills/litium-developer/` directory and registers it in your `skills-lock.json`.

### Install `litium-cloud-cli`

```bash
npx skills add https://github.com/LitiumAB/agent-skills --skill litium-cloud-cli
```

### Install `litium-cloud-migration`

```bash
npx skills add https://github.com/LitiumAB/agent-skills --skill litium-cloud-migration
```

Install `litium-cloud-cli` as well: the migration skill delegates all command syntax to it.

## Sample Usage

Once installed, the skill is automatically available to your AI agent. You can invoke it conversationally:

**litium-developer:**

```
Set up a new Litium project with the React Accelerator.
```

```
Create a new storefront page component with GraphQL data fetching.
```

```
Add a custom field type and field template for products.
```

```
Create a new Litium backoffice extension called "product-labels" using React.
```

**litium-cloud-cli:**

```
Create a test environment and install Litium platform, CDN and Insights in it.
```

```
Set up a service principal and an Azure DevOps pipeline that deploys our .NET artifact.
```

**litium-cloud-migration:**

```
We have a Litium 8.14 MVC site on legacy cloud with SFTP integration to an ERP and Klarna. Help me plan the move to Serverless Cloud.
```

```
Write the go-live runbook for next Thursday. The customer is already on Fastly.
```

The agent will follow the skill's instructions — scaffolding projects, generating code, explaining patterns, and guiding deployments — without any extra configuration.
