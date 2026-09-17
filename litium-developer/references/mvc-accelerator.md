# MVC Accelerator Development Guide

Development reference for the Litium MVC Accelerator — a .NET/ASP.NET Core e-commerce website.

## Overview

The MVC Accelerator is a full-featured ASP.NET Core web application with server-rendered views and optional headless React integration.

- Official docs: https://docs.litium.dev/accelerators/mvc-accelerator/install-litium-accelerator
- For local setup: see `references/setup-mvc-accelerator.md`

## Project Structure

```
Src/
├── Litium.Accelerator.Mvc/         # Main ASP.NET Core project
│   ├── Controllers/                # MVC Controllers
│   ├── Views/                      # Razor views (.cshtml)
│   ├── client/                     # TypeScript/SCSS source
│   │   ├── Scripts/                # TypeScript files
│   │   └── Styles/                 # SCSS files
│   └── wwwroot/ui/                 # Built JS/CSS (yarn prod output)
├── Litium.Accelerator/             # Core accelerator services / view models
│   ├── Search/                     # Search indexing and queries
│   ├── ViewModels/                 # View model classes
│   └── Builders/                   # View model builder services
├── Litium.Accelerator.Elasticsearch/  # Elasticsearch indexing
└── Litium.Accelerator.Administration.Extensions/ # Back office Angular extensions
```

## Key Patterns

### Controllers

MVC controllers extend `ControllerBase` and handle route requests:

```csharp
public class ProductController : ControllerBase
{
    private readonly ProductViewModelBuilder _viewModelBuilder;

    public ProductController(ProductViewModelBuilder viewModelBuilder)
    {
        _viewModelBuilder = viewModelBuilder;
    }

    [HttpGet]
    public ActionResult Index(ProductPage currentPage)
    {
        var model = _viewModelBuilder.Build(currentPage);
        return View(model);
    }
}
```

### View Models

View models are built via builder services and injected into Razor views:

```csharp
[Service(ServiceType = typeof(ProductViewModelBuilder))]
public class ProductViewModelBuilder
{
    public ProductViewModel Build(ProductPage page)
    {
        return new ProductViewModel { /* ... */ };
    }
}
```

### Razor Views

Views use strongly-typed models populated by builders:

```html
@model Litium.Accelerator.ViewModels.ProductViewModel
<div class="product">
    <h1>@Model.Name</h1>
    <p>@Model.Description</p>
</div>
```

### Dependency Injection

Register services using the `[Service]` attribute from `Litium.Runtime.DependencyInjection`:

```csharp
[Service(ServiceType = typeof(IMyService), Lifetime = DependencyLifetime.Scoped)]
public class MyService : IMyService { }
```

### Autostart Services

Run code at application startup:

```csharp
[Autostart]
public class MyStartupService : IAsyncAutostart
{
    public ValueTask StartAsync(CancellationToken cancellationToken)
    {
        // runs on startup
        return ValueTask.CompletedTask;
    }
}
```

## Client-Side Development

The MVC Accelerator uses Webpack + TypeScript for client-side code.

```powershell
# Development build with source maps
yarn run dev

# Production build (minified)
yarn run prod

# Watch mode (auto-rebuild on file changes — use during development)
yarn run watch
```

TypeScript entry files are in `client/Scripts/`. Output goes to `wwwroot/ui/`.

## React Components in MVC Views

The MVC Accelerator supports embedding React components in Razor views through the Extensions module.

Components are registered in `Litium.Accelerator.Administration.Extensions` and rendered via Angular/React widgets in the back office UI, or via custom TypeScript integration in MVC views.

**Example:** To render a React component within an MVC Razor view, add a container element and initialize via the client-side TypeScript entry point in `client/Scripts/`:

```html
<!-- In a Razor view -->
<div id="my-react-widget" data-product-id="@Model.ProductId"></div>
```

```typescript
// In client/Scripts/
import React from 'react';
import { createRoot } from 'react-dom/client';
import { MyWidget } from './Components/MyWidget';

const el = document.getElementById('my-react-widget');
if (el) {
    const root = createRoot(el);
    root.render(<MyWidget productId={el.dataset.productId} />);
}
```

For full headless React storefront integration, see `references/setup-react-accelerator.md`.

## Customizing Search (Elasticsearch)

MVC uses its own Elasticsearch indices (separate from the Storefront API):

```csharp
// Customize product indexing
[Service(ServiceType = typeof(IProductDocumentBuilder))]
public class CustomProductDocumentBuilder : ProductDocumentBuilderBase
{
    public override void Build(ProductDocument document, BaseProduct baseProduct)
    {
        base.Build(document, baseProduct);
        // Add custom fields to the index document
        document.Fields["customField"] = baseProduct.Fields.GetValue<string>("myCustomField");
    }
}
```

Elasticsearch index classes: `Litium.Accelerator.Elasticsearch/`

## Field Templates in MVC

Field templates define the structure of editable content in Litium. They are typically created programmatically in an `IAsyncAutostart` service:

```csharp
var template = new ProductFieldTemplate("MyTemplate")
{
    ProductFieldGroups = new[]
    {
        new FieldTemplateFieldGroup
        {
            Id = "General",
            Fields = { "MyField1", "MyField2" }
        }
    }
};
_fieldTemplateService.Create(template);
```

See `references/data-modelling.md` for full data modelling guidance.

## Useful Links

- MVC install guide: https://docs.litium.dev/accelerators/mvc-accelerator/install-litium-accelerator
- Creating custom blocks in MVC: https://docs.litium.dev/platform/guides/how-to-create-custom-blocks-in-mvc-accelerator
- Back office UI extensions: https://docs.litium.dev/platform/guides/back-office-ui-extensions
- Data modelling: https://docs.litium.dev/platform/guides/data-modelling/overview
- Elasticsearch: https://docs.litium.dev/platform/guides/litiumsearch/overview
