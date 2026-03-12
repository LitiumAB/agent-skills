# Empty Project Setup Guide

Complete guide for setting up a Litium Empty Project local environment.

> **Version note:** This guide uses Litium 8.28.0 as an example. Replace version numbers with the latest available version from the [Litium NuGet feed](https://nuget-release.litium.com/nuget/). Check with `dotnet new search litium --nuget-source https://nuget-release.litium.com/nuget/`.

## Purpose

Bare Litium installation without accelerator UI. Useful for:
- Custom implementations without pre-built UI
- Minimal backend for React Accelerator development
- Testing core Litium platform functionality

## Prerequisites

```bash
pwsh --version        # PowerShell Core 7.0+
dotnet --version      # .NET SDK 8.0+
node --version        # Node.js 20+ (if using React)
```

Required services (use Docker Compose — see https://litium.mintlify.app/platform/get-started/shared-dependencies):
- SQL Server on localhost:1433 (sa / Pass@word)
- Elasticsearch on http://localhost:9200

## Quick Start

Official docs: https://litium.mintlify.app/platform/get-started/install-empty-litium

### Step 1: Add Litium NuGet Source

```powershell
dotnet nuget add source https://nuget-release.litium.com/nuget/ --name LitiumRelease
```

### Step 2: Install Template

```powershell
dotnet new install Litium.Empty.Templates::8.28.0 --nuget-source https://nuget-release.litium.com/nuget/
```

### Step 3: Create Project

```powershell
# Create a folder and scaffold the project
mkdir litium-installations/EmptyProject_8_28_0
cd litium-installations/EmptyProject_8_28_0

# With Storefront API (recommended if using React)
dotnet new litemptyweb --storefront-api

# Without Storefront API (MVC/custom only)
dotnet new litemptyweb
```

### Step 4: Create Shared Folder

```powershell
mkdir ../../Shared
```

### Step 5: Configure appsettings.Development.json

Create `appsettings.Development.json` in the project folder:

```json
{
  "Litium": {
    "Data": {
      "ConnectionString": "Server=localhost,1433;Initial Catalog=LitiumEmpty_8280;User ID=sa;Password=Pass@word;TrustServerCertificate=True;"
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
dotnet tool restore
dotnet restore
dotnet litium-db update
dotnet litium-db user --login admin --password Password!
```

### Step 7: Install Storefront CLI (if using React)

```powershell
dotnet tool update -g Litium.Storefront.Cli
```

## Start the Site

```bash
# macOS / Linux
ASPNETCORE_ENVIRONMENT=Development dotnet run --urls "https://localhost:5001"

# Windows PowerShell
$env:ASPNETCORE_ENVIRONMENT="Development"; dotnet run --urls "https://localhost:5001"
```

**Back Office:** https://localhost:5001/Litium/  
Login: `admin` / `Password!`

## Folder Structure

```
litium-installations/EmptyProject_8_28_0/
├── Shared/                           # File storage (created separately)
├── *.csproj
├── Program.cs
├── appsettings.json
├── appsettings.Development.json      # Your local config
└── .config/
    └── dotnet-tools.json
```

## Common Issues

### Template not found
```powershell
# Verify NuGet source configured
dotnet nuget list source

# Add if missing
dotnet nuget add source https://nuget-release.litium.com/nuget/ --name LitiumRelease
```

### SQL Server connection fails
```powershell
# Test connection
sqlcmd -S localhost,1433 -U sa -P "Pass@word" -Q "SELECT 1"

# Check Docker container running
docker ps | grep sql
```

For more troubleshooting, see `references/setup-troubleshooting.md`.

## Version Compatibility

| Litium Version | .NET SDK |
|----------------|----------|
| 8.28.0+ | 8.0+ |
| 8.27.0  | 8.0+ |

## Next Steps

- **For custom development**: Start building controllers, views, business logic
- **For React backend**: Proceed with `references/setup-react-accelerator.md`
