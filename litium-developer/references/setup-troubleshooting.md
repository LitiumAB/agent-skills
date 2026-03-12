# Local Environment Troubleshooting

Common issues when setting up Litium installations locally.

## Template Installation

### Package not found

```powershell
# Check NuGet sources
dotnet nuget list source

# Add Litium release feed if missing
dotnet nuget add source https://nuget-release.litium.com/nuget/ --name LitiumRelease
```

### Template already installed

```powershell
dotnet new uninstall Litium.Empty.Templates
dotnet new uninstall Litium.Accelerator.Templates
dotnet new uninstall Litium.Accelerator.React.Templates
```

Then retry installation.

---

## Database Connection

### "A network-related or instance-specific error" — SQL Server unreachable

Use Docker Compose for easy service management: https://litium.mintlify.app/platform/get-started/shared-dependencies

```powershell
# Check Docker SQL container
docker ps | grep sql

# Test connection
sqlcmd -S localhost,1433 -U sa -P "Pass@word" -Q "SELECT @@VERSION"
```

If using a non-default port, update the `Server=` value in your connection string (e.g., `Server=localhost,5434`).

### Login failed for user 'sa'

- SQL Server must allow SQL authentication (not Windows-only)
- Check Docker `SA_PASSWORD` environment variable: `docker inspect <containerid> | grep SA_PASSWORD`

### Database already exists

- When prompted by the setup, answer `y` to drop and recreate
- Or manually: `DROP DATABASE [LitiumMvc_8280]`

---

## .NET Package Restore

### NuGet package restore failed

```powershell
dotnet nuget locals all --clear
dotnet restore --force
```

### Firewall / proxy issues

Configure dotnet to use your proxy settings if behind a corporate firewall.

---

## Database Schema Update

### litium-db update fails

```powershell
# Test database is accessible
sqlcmd -S localhost,1433 -U sa -P "Pass@word" -d LitiumMvc_8280 -Q "SELECT COUNT(*) FROM INFORMATION_SCHEMA.TABLES"
```

Ensure the connection string in `appsettings.Development.json` matches the database name you created.

---

## Client Builds (MVC)

### yarn install fails

```bash
node --version   # Must be 20+
yarn --version   # Must be installed

# Clean and retry
yarn cache clean
rm -rf node_modules
# Windows PowerShell: Remove-Item -Recurse -Force node_modules
yarn install --ignore-engines
```

### yarn run prod fails (TypeScript/Webpack errors)

```bash
rm -rf node_modules
# Windows PowerShell: Remove-Item -Recurse -Force node_modules
yarn install --ignore-engines
yarn run prod
```

Check output for exact error. Node version mismatches and missing dependencies are common causes.

---

## Certificates

### "NET::ERR_CERT_AUTHORITY_INVALID" in browser

```powershell
dotnet dev-certs https --clean
dotnet dev-certs https --trust
```

### Certificate errors in React / Storefront CLI

Add to `.env.local`:
```env
NODE_TLS_REJECT_UNAUTHORIZED="0"
```

Or add `--insecure` to `litium-storefront` commands.

---

## Port Conflicts

| Port | Service | Fix |
|------|---------|-----|
| 1433 | SQL Server | Stop other SQL instances or use `Server=localhost,5434` |
| 5001 | Litium Backend | Stop other ASP.NET apps |
| 3000 | React Dev Server | Edit `package.json` dev script |
| 3001 | Storefront Proxy | Use `--port 3002` flag |
| 9200 | Elasticsearch | Stop other ES instances |

```bash
# Find what's using a port
# macOS / Linux:
lsof -i :5001
# Windows:
# netstat -ano | findstr :5001
```

---

## React / Next.js

### Definition import fails

```powershell
# Verify backend running
curl https://localhost:5001/Litium/ --insecure

# Run import manually
litium-storefront definition import \
  --file "litium-definitions/**/*.yaml" \
  --litium https://localhost:5001 \
  --litium-username admin \
  --litium-password "Password!" \
  --insecure
```

### React shows 404 for all pages

In Back Office: Websites → Edit website → Settings tab:
- Set **External storefront** = `headless-accelerator`
- Add **Domain**: `localhost`

### GraphQL errors in browser console

1. Verify Storefront API is enabled on backend
2. Check schema accessible: https://localhost:5001/storefront.graphql?sdl
3. Re-run field definitions import

### npm install fails

```bash
npm cache clean --force
rm -f package-lock.json
rm -rf node_modules
# Windows PowerShell: Remove-Item -Force package-lock.json; Remove-Item -Recurse -Force node_modules
npm install
```

---

## MVC Demo Site

### Accelerator not appearing after deployment

1. Back Office → Settings → Deployment → Accelerator — check deployment completed
2. Back Office → System → Clear cache
3. Restart application

### Site has no styling / missing assets

Client build artifacts (JS/CSS) may be missing. Run in `Src/Litium.Accelerator.Mvc`:

```bash
yarn install --check-files && yarn run prod
```

Verify `wwwroot/ui/` exists with `js/` and `css/` subfolders.

---

## Elasticsearch

### "Elasticsearch cluster not available"

Use Docker Compose: https://litium.mintlify.app/platform/get-started/shared-dependencies

```powershell
curl http://localhost:9200    # Should return cluster health JSON
docker start elasticsearch    # Start if stopped
```

---

## Cleanup

### Remove a single installation database

```bash
sqlcmd -S localhost,1433 -U sa -P "Pass@word" -Q "DROP DATABASE [LitiumMvc_8280]"
rm -rf ./litium-installations/MvcAccelerator_8_28_0
# Windows PowerShell: Remove-Item -Recurse -Force ./litium-installations/MvcAccelerator_8_28_0
```

### Full reset

```powershell
# Stop all running processes (Ctrl+C)

# Drop all Litium test databases
sqlcmd -S localhost,1433 -U sa -P "Pass@word" -Q "
SELECT 'DROP DATABASE [' + name + ']' FROM sys.databases WHERE name LIKE 'Litium%'"
# Review the output, then copy and execute each DROP statement

# Remove project folders
rm -rf ./litium-installations
# Windows PowerShell: Remove-Item -Recurse -Force ./litium-installations

# Clear NuGet template cache
dotnet nuget locals all --clear
dotnet new uninstall Litium.Empty.Templates
dotnet new uninstall Litium.Accelerator.Templates
dotnet new uninstall Litium.Accelerator.React.Templates
```

---

## Documentation References

- Empty Project setup: https://litium.mintlify.app/platform/get-started/install-empty-litium
- MVC Accelerator setup: https://litium.mintlify.app/accelerators/mvc/install-litium-accelerator
- React Accelerator setup: https://litium.mintlify.app/accelerators/react/get-started
- Shared dependencies (Docker Compose): https://litium.mintlify.app/platform/get-started/shared-dependencies
