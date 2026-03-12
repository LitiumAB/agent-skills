---
name: litium-developer
description: "Help Litium partner developers build, configure, and deploy e-commerce websites using the Litium platform. Use for React Accelerator (Next.js, GraphQL, blocks, cart, checkout, wishlist), MVC Accelerator (.NET Razor views, controllers, view models, client builds), local environment setup (empty project, MVC Accelerator, React Accelerator), deploy to Litium Cloud, custom field types, field templates, multi-field, data modelling, Admin Web API, Storefront GraphQL API, Connect ERP/Payment/Shipment API, Extension Management API, and searching forum.litium.com for community solutions. Triggers on: build Litium site, create Litium project, add product fields, create block, set up locally, configure accelerator, Litium API, wishlist, cart mutation, placeOrder, custom field."
---

# Litium Developer

## Overview

Partner developers building on Litium platform use this skill for implementation guidance. This skill delegates to the `litium-cloud-cli` sub-skill for cloud deployment. For all other Litium topics — including React Accelerator, MVC, data modelling, and APIs — it loads reference files on demand.

Always read `references/partner-guidance.md` first before helping modify Litium platform files.

## Delegation

For this area, delegate entirely to the named skill — do not duplicate its content here:

| Topic | Skill to use |
|-------|-------------|
| Deploy to Litium Cloud, create/manage environments, artifacts, CI/CD, YAML apply manifests, service principals | `litium-cloud-cli` |

## Reference map

Read these files **only** when the user's task requires that area:

| Reference file | Load when the user asks about |
|----------------|-------------------------------|
| `references/partner-guidance.md` | what partners can/cannot modify, correct extension patterns, internal API disclaimer, protected projects, NuGet feeds, Elasticsearch index separation |
| `references/setup-empty-project.md` | set up bare Litium, empty project, install bare platform, download Litium locally, `litemptyweb` template |
| `references/setup-mvc-accelerator.md` | install MVC Accelerator locally, `litmvcacc` template, yarn build, deploy accelerator in back office |
| `references/setup-react-accelerator.md` | install React Accelerator locally, `litreactacc` template, storefront proxy, `.env.local`, headless accelerator |
| `references/setup-troubleshooting.md` | setup error, local environment issue, install fails, SQL connection, certificate, port conflict, package restore |
| `references/mvc-accelerator.md` | MVC development, .NET controller, Razor view, view model, webpack client build, MVC + React component, Elasticsearch MVC |
| `references/data-modelling.md` | custom field types, create field type, field template, folder template, product template, multi-field, field not in template, `FieldTypeMetadataBase` |
| `references/cart-and-wishlist.md` | cart, checkout, order, wishlist, `addVariantToCart`, `createCheckoutSession`, `placeOrder`, `CartContext`, `BuyButton`, payment widget |
| `references/apis.md` | Admin Web API, Storefront GraphQL API, Connect ERP API, Payment API, Shipment API, Extension Management API, OpenAPI spec |
| `references/forum-api.md` | search forum.litium.com, community solution, Discourse API, find forum post |
| `references/react-accelerator/overview.md` | React Accelerator development, Next.js architecture, server vs. client components, GraphQL patterns, component conventions, testing |
| `references/react-accelerator/template-routing.md` | create page/product/category/block template, proxy routing, `[[...slug]]` catch-all, template naming conventions |
| `references/react-accelerator/code-patterns.md` | React Accelerator code patterns, `queryServer`, `mutateClient`, service layers, fragments, Apollo Client usage |
| `references/react-accelerator/project-structure.md` | React Accelerator directory layout, `app/`, `components/`, `services/`, `operations/` folders |
| `references/react-accelerator/graphql-sdl.md` | fetch Storefront GraphQL SDL, schema download, `RUNTIME_LITIUM_SERVER_URL`, available operations |

## Documentation search priority

1. **Primary**: https://litium.mintlify.app/ — always try first for official docs.
2. **Fallback**: https://docs.litium.com/ — use if Mintlify search returns no result.
3. **Community**: https://forum.litium.com/ — for real-world solutions; load `references/forum-api.md` to search programmatically.

## Quick-start routing

**Step 0 (always):** Read `references/partner-guidance.md` first to understand what partners can/cannot modify.

- **"How do I work on the React storefront?"** → read `references/react-accelerator/overview.md` first, then load specific references as needed (template-routing, code-patterns, etc.).
- **"How do I deploy/manage Litium Cloud environments?"** → delegate to `litium-cloud-cli`.
- **"How do I set up Litium locally?"** → load the matching setup reference (empty / MVC / React).
- **"How do I add cart or wishlist functionality?"** → load `references/cart-and-wishlist.md`.
- **"How do I create a custom field type or field template?"** → load `references/data-modelling.md`.
- **"How do I call the Admin Web API or a Connect API?"** → load `references/apis.md`.
