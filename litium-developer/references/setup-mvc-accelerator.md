# MVC Accelerator Setup Guide

Complete guide for setting up a Litium MVC Accelerator local environment.

> **Version note:** This guide uses Litium 8.28.0 as an example. Replace version numbers with the latest available version from the [Litium NuGet feed](https://nuget.litium.com/nuget/). Check with `dotnet new search litium --nuget-source https://nuget.litium.com/nuget/`.

## Purpose

Full MVC Accelerator with demo site and Storefront API. Used for:
- .NET e-commerce backend development
- Backend for React Accelerator
- Development and customization of MVC-based storefronts

## Prerequisites

```bash
pwsh --version        # PowerShell Core 7.0+
dotnet --version      # .NET SDK 8.0+
node --version        # Node.js 20+
yarn --version        # Yarn (for client builds)
```

Required services (use Docker Compose — see https://docs.litium.dev/platform/get-started/shared-dependencies/overview):
- SQL Server on localhost:1433 (sa / Pass@word)
- Elasticsearch on http://localhost:9200

## Quick Start

Official docs: https://docs.litium.dev/accelerators/mvc-accelerator/install-litium-accelerator

### Step 1: Add Litium NuGet Source

```powershell
dotnet nuget add source https://nuget.litium.com/nuget/ --name LitiumRelease
```

### Step 2: Install Template

```powershell
dotnet new install Litium.Accelerator.Templates::8.28.0 --nuget-source https://nuget.litium.com/nuget/
```

### Step 3: Create Project

```powershell
mkdir litium-installations/MvcAccelerator_8_28_0
cd litium-installations/MvcAccelerator_8_28_0

# With Storefront API (recommended for React integration)
dotnet new litmvcacc --storefront-api

# Without Storefront API (MVC-only)
dotnet new litmvcacc
```

### Step 4: Create Shared Folder

```powershell
mkdir Shared
```

### Step 5: Configure appsettings.Development.json

Create `Src/Litium.Accelerator.Mvc/appsettings.Development.json`:

```json
{
  "Litium": {
    "Data": {
      "ConnectionString": "Server=localhost,1433;Initial Catalog=LitiumMvc_8280;User ID=sa;Password=Pass@word;TrustServerCertificate=True;"
    },
    "Folder": {
      "Local": "../../Shared",
      "Shared": "../../Shared"
    },
    "Elasticsearch": {
      "ConnectionString": "http://localhost:9200"
    },
    "Websites": {
      "Storefronts": {
        "headless-accelerator": {
          "host": "https://localhost:3001"
        }
      }
    }
  }
}
```

### Step 6: Restore and Update Database

```powershell
cd Src/Litium.Accelerator.Mvc
dotnet tool restore
dotnet restore
dotnet litium-db update
dotnet litium-db user --login admin --password Password!
```

### Step 7: Build Client Projects

This is the longest step (~5-7 minutes).

```powershell
# Build Extensions module
cd ../Litium.Accelerator.Administration.Extensions
yarn install && yarn run prod

# Build Email project (if exists)
cd ../Litium.Accelerator.Email
yarn install && yarn run prod

# Build MVC client
cd ../Litium.Accelerator.Mvc
yarn install --check-files && yarn run prod
```

### Step 8: Install Storefront CLI (if using React)

```powershell
dotnet tool update -g Litium.Storefront.Cli
```

## Start the Site

```bash
cd Src/Litium.Accelerator.Mvc

# macOS / Linux
ASPNETCORE_ENVIRONMENT=Development dotnet run --urls "https://localhost:5001"

# Windows PowerShell
$env:ASPNETCORE_ENVIRONMENT="Development"; dotnet run --urls "https://localhost:5001"
```

**Back Office:** https://localhost:5001/Litium/  
Login: `admin` / `Password!`

## Deploy Accelerator (Required!)

The demo site won't display content until you deploy the Accelerator:

1. In Back Office, go to: **Settings → Deployment → Accelerator**
2. Enter a name (e.g., "My Site") and domain: `localhost`
3. Click **Import** — wait 1-2 minutes
4. Navigate to https://localhost:5001 — you should see the demo site

## Folder Structure

```
litium-installations/MvcAccelerator_8_28_0/
├── Shared/
├── Src/
│   ├── Litium.Accelerator.Mvc/
│   │   ├── appsettings.Development.json
│   │   ├── Controllers/, Views/
│   │   ├── client/               # TypeScript/SCSS source
│   │   └── wwwroot/ui/           # Built JS/CSS (after yarn prod)
│   ├── Litium.Accelerator.Email/
│   └── Litium.Accelerator.Administration.Extensions/
└── .config/dotnet-tools.json
```

## Client Build Commands

```powershell
yarn run prod    # Production build (minified)
yarn run dev     # Development build (source maps)
yarn run watch   # Watch mode (auto-rebuild on changes)
```

## Common Issues

For troubleshooting client build failures, Accelerator deployment issues, SQL/Elasticsearch connection problems, and other common issues, see `references/setup-troubleshooting.md`.

## Version Compatibility

| Litium Version | .NET SDK | Node.js |
|----------------|----------|---------|
| 8.28.0+ | 8.0+ | 20+ |
| 8.27.0  | 8.0+ | 20+ |

## Next Steps

- **Deploy Accelerator** — required before demo site works
- **Set up React Accelerator** — see `references/setup-react-accelerator.md`
- **Start developing** — see `references/mvc-accelerator.md`
