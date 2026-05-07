# litium-extension-dev

A skill for building, maintaining, and deploying Litium backoffice extensions using `@litiumab/platform-extension-sdk`. Covers all four supported frameworks: React, Vue, Angular, and Vanilla JS.

## Install

```bash
npx skills add https://github.com/LitiumAB/agent-skills --skill litium-extension-dev
```

## What it does

- Scaffolds new extension projects with the `create` CLI command
- Adds panels and settings pages with the `add` CLI command
- Explains the `extension.manifest.json` deployment descriptor
- Covers the `window.litiumExtension` bridge API (navigate, notifications, getContext, events)
- Provides framework-specific patterns for React, Vue, Angular, and Vanilla JS
- Implements sub-path routing (MemoryRouter, createMemoryHistory, MemoryLocationStrategy)
- Guides IIFE bundle builds with the Litium Vite plugin
- Includes unit testing patterns with `window.litiumExtension` mocks
- Automates migration from Angular Module Federation to Web Components
- Deploys extensions via Settings > Extensions UI or the Extension Management API

## No configuration required
