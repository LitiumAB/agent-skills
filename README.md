# Litium Agent Skills

A collection of AI agent skills for building on the Litium platform. Works with GitHub Copilot, Cursor, Windsurf, and any other agent that supports the skills standard.

## Available Skills

| Skill | Description |
|-------|-------------|
| [litium-developer](./litium-developer/README.md) | Comprehensive development skill for Litium partner developers — covers React Accelerator (Next.js), MVC Accelerator (.NET), back office UI extensions, data modelling, APIs, local setup, and troubleshooting. |
| [litium-extension-dev](./litium-extension-dev/README.md) | Build, maintain, and deploy Litium backoffice UI extensions using `@litium/platform-extension-sdk`. Covers React, Vue, Angular, and Vanilla JS. |

## Installing a Skill

Skills are installed with the [`skills` CLI](https://www.npmjs.com/package/skills):

```bash
npx skills add https://github.com/LitiumAB/agent-skills --skill <skill-name>
```

### Install `litium-developer`

```bash
npx skills add https://github.com/LitiumAB/agent-skills --skill litium-developer
```

### Install `litium-extension-dev`

```bash
npx skills add https://github.com/LitiumAB/agent-skills --skill litium-extension-dev
```

This copies the skill files into your project's `.agents/skills/litium-extension-dev/` directory and registers it in your `skills-lock.json`.

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

**litium-extension-dev:**

```
Create a new Litium backoffice extension called "product-labels" using React.
```

```
Migrate my existing Angular Module Federation extension to the new Web Component format.
```

The agent will follow the skill's instructions — scaffolding projects, generating code, explaining patterns, and guiding deployments — without any extra configuration.
