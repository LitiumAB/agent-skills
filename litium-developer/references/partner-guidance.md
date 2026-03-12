# Partner Developer Guidance

Key rules and patterns for Litium partner developers building on the Litium platform.

## ⛔ Protected Projects — DO NOT Modify

These projects are delivered as binaries. Partners cannot modify them:

- `Litium.Storefront.Application` (GraphQL Storefront API)
- `Litium.Web.Application` (Admin Web API)
- `Litium.Application` (Platform services)
- All `Litium/Src/` projects except accelerator projects
- All Litium Apps (`Litium.Apps.DirectShipment`, `Litium.Apps.KlarnaPayment`, etc.)

> **NEVER suggest adding mutations/endpoints to protected projects.**

## ✅ Partner-Modifiable Projects

These are delivered as source code and can be customized:

- `Litium.Storefront` — React Accelerator (Next.js)
- `Litium.Accelerator.Mvc` — MVC Accelerator
- `Litium.Accelerator` — Core accelerator framework
- `Litium.Accelerator.Administration.Extensions` — Back office UI extensions

## Creating Custom Apps

Partners **can** create new payment/shipment apps using the SDKs:

- Custom payment integrations → use `Litium.Apps.Payment` SDK
- Custom shipment integrations → use `Litium.Apps.Shipment` SDK
- App Development Guide: https://litium.mintlify.app/apps/development-guide/overview

## Correct Extension Patterns

### Extending the Storefront API

GraphQL Federation is not yet available. The correct pattern is:

1. Create a Next.js API route in the React Accelerator:
   ```typescript
   // app/api/register/route.ts
   export async function POST(request: Request) {
     const data = await request.json();
     const response = await fetch('https://domain.com/litium/api/admin/customers/people', {
       method: 'POST',
       headers: {
         'Content-Type': 'application/json',
         'Authorization': `Bearer ${token}`
       },
       body: JSON.stringify(data)
     });
     return Response.json(await response.json());
   }
   ```
2. Frontend calls your custom Next.js API route

### .NET Development — Priority Order

1. **Prefer REST/GraphQL APIs** — Admin Web API or Storefront API (best maintainability)
2. **Fallback to .NET services** — Use `*.Abstractions` assemblies if APIs can't solve the problem
   - Example: `PersonService.Create()` from `Litium.Abstractions`
   - These have backward compatibility guarantees
3. **Avoid `*.Application` assemblies** — Internal, no backward compatibility guarantee

### Back Office UI Extensions

- Use the Litium Extensions system (see `references/extensions/core-rules.md`)
- Use `Litium.Client.UI` npm package from `packages.litium.com/Npm/`
- See: https://litium.mintlify.app/platform/guides/back-office-ui-extensions

## Public vs. Internal API Namespaces

### Public APIs (no disclaimer needed)
- `Litium.Abstractions` namespace
- `Litium.Accelerator*` namespaces

### Internal APIs (disclaimer required)
- `Litium.Application` namespace
- `Litium.Web.Application` namespace
- Any other `Litium.*` namespace not listed as public

### ⚠️ Required Disclaimer Format

When a solution requires internal APIs, always include:

```
> ⚠️ **Internal API Disclaimer**
>
> This solution uses internal Litium APIs that are **not part of the public API surface**.
> These APIs may change or be removed without notice in future Litium versions.
>
> **Internal APIs used:**
> - `Namespace.ClassName`
>
> Only `Litium.Abstractions` and `Litium.Accelerator*` namespaces are considered
> public APIs with backward compatibility guarantees.
```

After providing an internal API solution, always mention if there is a public API alternative or feature request opportunity.

## NuGet Packages

Public Litium NuGet feeds:
- **Release**: `https://nuget-release.litium.com/nuget/`
- **Production**: `https://nuget.litium.com/nuget/`
- **npm packages**: `https://packages.litium.com/Npm/`

Add the release feed:
```powershell
dotnet nuget add source https://nuget-release.litium.com/nuget/ --name LitiumRelease
```

## Elasticsearch — MVC vs. React Accelerator (Separate Indices)

| Component | Index Documents | Customizable by Partners |
|-----------|-----------------|--------------------------|
| MVC Accelerator | `ProductDocument`, `CategoryDocument`, `PageDocument` | ✅ Yes |
| Storefront API (React) | `StorefrontProductDocument`, `StorefrontCategoryDocument` | ❌ No (protected) |
| Backoffice | Uses database queries (DataService), not Elasticsearch | ✅ N/A |

**Customizing MVC indices does NOT affect Storefront API (React) search results.**

## Finding Documentation

1. **Primary**: https://litium.mintlify.app/
2. **Fallback**: https://docs.litium.com/
3. **Community forum**: https://forum.litium.com/ (see `references/forum-api.md` for API search)
