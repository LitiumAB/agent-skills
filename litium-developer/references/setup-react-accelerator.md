# React Accelerator Setup Guide

Complete guide for setting up the Litium React Accelerator (Next.js storefront).

> **Version note:** This guide uses React Accelerator 1.13.0 as an example. Replace version numbers with the latest available version. Check with `dotnet new search litium --nuget-source https://nuget.litium.com/nuget/`.

## Purpose

Modern headless storefront for Litium. Used for:
- Next.js / React frontend development
- Headless e-commerce storefront
- Integration with Litium Storefront API (GraphQL)

## Prerequisites

**Requires an existing Litium backend with Storefront API:**
- MVC Accelerator with Storefront API, OR
- Empty Project with Storefront API

Backend must be running at https://localhost:5001.

```bash
node --version    # 20+
npm --version     # 9+
dotnet tool list -g | grep storefront  # Litium.Storefront.Cli
```

## Quick Start

Official docs: https://docs.litium.dev/accelerators/react-accelerator/get-started

### Step 1: Add Litium NuGet Source (if not already done)

```powershell
dotnet nuget add source https://nuget.litium.com/nuget/ --name LitiumRelease
```

### Step 2: Install Template

```powershell
dotnet new install Litium.Accelerator.React.Templates::1.13.0 --nuget-source https://nuget.litium.com/nuget/
```

### Step 3: Create Project

```powershell
# Standalone (or nest inside MVC folder for integrated development)
mkdir litium-installations/ReactAccelerator_1_13_0
cd litium-installations/ReactAccelerator_1_13_0
dotnet new litreactacc
```

### Step 4: Configure .env.local

Create `.env.local` in the project root:

```env
RUNTIME_LITIUM_SERVER_URL=https://localhost:5001
NODE_TLS_REJECT_UNAUTHORIZED="0"
```

### Step 5: Install Storefront CLI

```powershell
dotnet tool update -g Litium.Storefront.Cli
```

### Step 6: Install Node.js Dependencies

```powershell
npm install
```

### Step 7: Import Field Definitions

Ensure the backend is running before this step:

```powershell
litium-storefront definition import \
  --file "litium-definitions/**/*.yaml" \
  --litium https://localhost:5001 \
  --litium-username admin \
  --litium-password "Password!" \
  --insecure
```

## Configure Back Office (Required!)

React won't work until the website is configured:

1. In Back Office, go to: **Websites → [your website] → Edit**
2. Open the **Settings** tab
3. Set **External storefront** to `headless-accelerator`
4. Add **Domain name**: `localhost`
5. Click **Save**

Also change the home page template:

> **Why?** The React Accelerator uses a different rendering pipeline. The default "Home page" template is MVC-specific and won't render correctly in the headless React storefront. "Landing page" is the template designed for the React Accelerator.

1. Go to **Websites → [website tree] → Home**
2. Click **Edit → Settings tab**
3. Change **Template** from "Home page" to **Landing page**
4. Click **Save** then **Publish**

## Start Development

Open three terminals:

```powershell
# Terminal 1: Backend (if MVC Accelerator)
cd litium-installations/MvcAccelerator_8_28_0/Src/Litium.Accelerator.Mvc
dotnet run

# Terminal 2: React dev server
cd litium-installations/ReactAccelerator_1_13_0
npm run dev   # or: yarn dev

# Terminal 3: Storefront proxy
litium-storefront proxy --litium https://localhost:5001 --storefront http://localhost:3000
```

**Access storefront:** https://localhost:3001  
**React dev server:** http://localhost:3000 (direct, no proxy)

## Architecture

```
Browser → https://localhost:3001
           ↓
    Storefront Proxy (:3001)
     ↙            ↘
React Next.js    MVC Backend
   (:3000)         (:5001)
                 └─ GraphQL API
                 └─ Admin REST API
```

## Project Structure

```
ReactAccelerator_1_13_0/
├── .env.local                  # Backend connection (your config)
├── app/                        # Next.js App Router pages
│   ├── layout.tsx
│   └── [...slug]/              # Dynamic Litium page routes
├── components/                 # React components
├── operations/                 # GraphQL operations (queries/mutations)
├── litium-definitions/        # Field definition YAML files
├── services/                   # API service layer
├── hooks/                      # React hooks
└── models/                     # TypeScript models
```

## Development Commands

```powershell
npm run dev      # Start dev server (hot reload)
npm run build    # Production build
npm run lint     # Run linting
```

## Common Issues

### Definition import fails

```powershell
# Verify backend is running
curl https://localhost:5001/Litium/ --insecure

# Run import manually with verbose
litium-storefront definition import \
  --file "litium-definitions/**/*.yaml" \
  --litium https://localhost:5001 \
  --litium-username admin \
  --litium-password "Password!" \
  --insecure
```

### React shows 404 for all pages

- Verify website configured with headless-accelerator storefront in Back Office
- Verify proxy is running and connecting to correct backend

### npm install fails

```bash
npm cache clean --force
rm -f package-lock.json
rm -rf node_modules
npm install
```

### GraphQL errors in console

1. Verify Storefront API is enabled on backend
2. Check schema: https://localhost:5001/storefront.graphql?sdl
3. Re-run field definitions import

For more troubleshooting, see `references/setup-troubleshooting.md`.

## Version Compatibility

| Litium Platform | React Accelerator | Node.js |
|-----------------|-------------------|---------|
| 8.28.0+ | 1.13.0+ | 20+ |
| 8.27.0  | 1.12.0  | 20+ |

## Next Steps

- **Working with React components**: see `references/react-accelerator/overview.md`
- **Cart and wishlist**: see `references/cart-and-wishlist.md`
- **Deploy to Litium Cloud**: see `litium-cloud-cli` skill
