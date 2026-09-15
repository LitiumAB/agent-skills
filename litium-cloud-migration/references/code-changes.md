# Code changes

Apply this top to bottom after the assessment (`references/assessment.md`) has produced the inventory. Every item below assumes you know, from that inventory, which packages, paths, configuration files, jobs and pipeline steps the solution has; this file says what to change them into. Record each change in `MIGRATION.md` under *Status*, and each thing you could not verify under *Open questions*.

Source of truth: https://docs.litium.dev/cloud/serverless/migration/prepare.md ("Make the solution ready for Linux") and https://docs.litium.dev/cloud/serverless/guides/artifacts/create-dotnet-artifact.md.

## 1. What "Linux-ready" means

The Litium platform app runs the published output of the web project in a Linux container, without IIS, without a Windows file system and without a `web.config`. The solution is Linux-ready when:

- `dotnet publish` with `--os linux -a x64` succeeds and the output contains `litiumcloud.manifest.json` and `license.json`;
- no package or code path requires Windows (section 3);
- every environment-specific value comes from configuration the platform injects or from the manifest (section 4);
- background jobs set their culture explicitly and no code assumes case-insensitive paths, a local time zone or files on disk that only exist on the legacy server (section 5);
- nothing the platform already provides is duplicated in the app (section 6).

Verify locally with the docs' publish command (section 2). If a Linux machine or container with the same .NET runtime is available, start the published output there and open the start page and the back office before creating the first artifact; a solution that only ever ran on Windows usually fails on paths or culture first. Do not rely on `dotnet run` on Windows as proof of anything in this file. The real verification is the test environment (`references/environment-setup.md`).

## 2. Build and packaging

### The Targets package and the cloud manifest

Add `Litium.Cloud.Tools.Targets` to the **startup web project** (the MVC project in the accelerator). It generates `litiumcloud.manifest.json` at publish time from the project's properties (entry point, target framework, Litium version, probe paths) and adds it to the published output. Without it the artifact cannot be run.

```bash
dotnet add ./Src/Litium.Accelerator.Mvc/Litium.Accelerator.Mvc.csproj package Litium.Cloud.Tools.Targets --prerelease
```

Do not hand-write the manifest, and do not check a generated one into the repo.

### Publish flags

Publish per project with the flags from the docs; adjust `-f` to the solution's target framework:

```bash
dotnet publish ./Src/Litium.Accelerator.Mvc/Litium.Accelerator.Mvc.csproj -f net9.0 -c Release -o publish -a x64 --os linux
```

- Remove any `<RuntimeIdentifier>win-*</RuntimeIdentifier>`, `<PlatformTarget>x86</PlatformTarget>` and `.pubxml` publish profiles found in the assessment; `--os linux -a x64` on the command line is the only runtime selection.
- The legacy pipeline publishes the whole solution (`dotnet publish *.sln`); publish the **web project** instead, because the artifact is that project's output folder. Any other project that must run in Serverless Cloud (an integration service, a scheduler host) becomes a private .NET web app with its own artifact: add the Targets package to that project too, publish it with the same flags, and see `references/environment-setup.md` for the app. A Windows Service or console worker has no app type of its own; host it as a background service inside the Litium platform app or a private .NET web app, and give it the health endpoints or `"none"` probes (section 6). Verify the hosting choice in the test environment.
- Run the client build (`yarn install` and `yarn run prod` in the folders the legacy pipeline lists, typically `Src`, `Src/Litium.Accelerator.Mvc` and `Src/Litium.Accelerator.Email`) **before** `dotnet publish`, exactly as today; the publish output must contain the built client assets.

### license.json

`license.json` must be inside the published output; the site shows a license error otherwise. If the legacy release copies it from a secure file in the pipeline, keep that step and copy it into the `publish` folder before `artifact create`. If it is in the repo, confirm it is set to copy to output. The license is bound to the domain, so the test environment needs a license that covers the test domain; ask the customer or Litium support where that comes from.

### .litiumcloudignore and artifact contents

`artifact create --file-path ./publish` reads `.gitignore`, `.npmignore` and `.litiumcloudignore` **from the root of the uploaded folder only**, in that order; the last matching rule wins, `!pattern` re-includes, and `.git/**` is always excluded. The publish folder normally has none of these, so everything in it is uploaded. Add a `.litiumcloudignore` in the publish folder only when a file that must be in the artifact is excluded (for example a copied `.gitignore` that hides `*.json`), or when build leftovers should be excluded.

Into the artifact: the publish output, `license.json`, `appsettings.json`, `appsettings.production.json`, `nlog.config`, built client assets, `litiumcloud.manifest.json`. Not into the artifact: source, `node_modules`, test output, `appsettings.<Env>.json` files, `web.config` (harmless but unused), `.pubxml`, media, database backups, secrets and certificates. Check the size before the first upload; anything over a few hundred MB means the wrong folder was packaged. Inspect a packaged artifact with `litium-cloud artifact download`.

## 3. Windows-only dependencies

Each assessment hit maps to one row. A replacement in a core flow (image processing, PDF, barcodes) is development work; do it before the test environment.

| Found | Why it fails on Linux | Replace with |
|---|---|---|
| `System.Drawing.Common`, `System.Drawing` types in code | GDI+ is not available on Linux from .NET 6 on; throws `PlatformNotSupportedException` at first use | `SixLabors.ImageSharp` or `SkiaSharp` for resizing and drawing; Litium's own media and image APIs for thumbnails and image formats already handled by the platform (`litium-developer`). Delete the dependency, do not set the `EnableUnixSupport` switch |
| `Microsoft.Web.Administration` | Manages IIS | Delete. Recycling, app pools and site bindings do not exist; a restart is `app restart` (`litium-cloud-cli`) |
| `System.DirectoryServices*`, Windows authentication, `AllowWindowsCredential` in `Litium:AdministrationSecurity` | Active Directory and Windows accounts are not reachable | Litium Accounts sign-in and Litium users; for federated staff login use the OIDC options of Litium (`litium-developer`). Remove the settings |
| `Microsoft.Windows.Compatibility` | Meta-package that drags in every Windows-only API | Remove and add only the cross-platform packages actually used |
| `System.Management`, WMI, performance counters, `EventLog` | Windows-only | Delete; log through the normal logger (goes to Litium Insights), read resource use in Litium Insights |
| `Microsoft.Win32.Registry`, `Registry.*` | No registry | Configuration values (section 4) |
| `System.ServiceProcess`, `ServiceController` | No Windows services | Background service in the app, or a private app |
| `[DllImport]` of a Windows DLL, COM interop, `Process.Start` of `.exe` tools (ImageMagick, wkhtmltopdf, Office) | Binary is not present or not executable | A managed package, or an external service. A native Linux binary shipped in the artifact must be verified in a test environment; nothing installs it for you |
| `Litium:ThumbnailGenerator:BrowserExecutablePath` | A local Chrome path | Remove; the docs state no support in Serverless Cloud, thumbnails are handled by the platform |
| Absolute paths (`C:\`, `D:\`), UNC paths (`\\server\share`), backslash literals, `Path.Combine("C:", ...)` | Do not exist, and `\` is a normal character in a Linux file name | Paths from configuration (section 4), forward slashes or `Path.Combine` with segments only, `Path.DirectorySeparatorChar` where a separator is needed |
| File names that differ from the code only in case (`Views/Home/index.cshtml` vs `Index.cshtml`, `Fonts/Arial.ttf` vs `arial.ttf`) | Linux file systems are case sensitive; `File not found` at runtime | Make every path in code, views, configuration and CSS match the file on disk exactly; rename files rather than code where git history matters (`git mv`) |
| `Encoding.Default`, `Encoding.GetEncoding(1252)` and other code pages | `Encoding.Default` is UTF-8 on Linux; code pages need `System.Text.Encoding.CodePages` registered | Use `Encoding.UTF8` or `Encoding.Latin1` explicitly; for legacy ERP files register the code-page provider at startup and name the encoding explicitly |
| `web.config` `<rewrite>` rules | No IIS; the file is not read | ASP.NET Core rewrite or redirect middleware in the app, or Litium redirects in the back office; CDN-level rules are not self-service (ask support, `references/support-requests.md`) |
| `maxAllowedContentLength`, `maxRequestLength` | Not read | The request body limit for uploads is 100 MB and is not configurable; chunk larger uploads in the app. Artifact uploads are exempt (the CLI chunks them) |
| `customHeaders`, HSTS and security headers in `web.config` | Not read | Middleware in the app; see section 6 for HTTPS and HSTS behind the CDN |
| IP restrictions, basic authentication in `web.config` | Not read | Application code; a Litium SFTP allow list for file transfer |

## 4. Configuration

### Which files are loaded

The Litium platform app loads **only `appsettings.json` and `appsettings.production.json`**, in every environment. `appsettings.Staging.json`, `appsettings.Test.json`, Magic Chunk or XDT transforms and pipeline variable substitution have no effect, and `ASPNETCORE_ENVIRONMENT` does not make another `appsettings.<name>.json` load on a Litium platform app. (A private .NET web app of your own can use `ASPNETCORE_ENVIRONMENT` to load an extra `appsettings.<EnvironmentName>.json`, as in any ASP.NET Core app, see https://docs.litium.dev/cloud/serverless/guides/configure/app-configuration-and-secrets.md.) **Never set `ASPNETCORE_ENVIRONMENT=Development`**; the app does not start.

### What to move where

| Legacy value | Serverless Cloud | Code change |
|---|---|---|
| Any value that differs per environment and is not secret (ERP host, feature flag, external URL) | Manifest `configurations` entry with a plain `value`; the name uses `__` between levels, so `Litium:Accelerator:Erp:Host` becomes `LITIUM__ACCELERATOR__ERP__HOST` (names are upper-cased) | None; read it as today through `IConfiguration` |
| Passwords, API keys, tokens | Environment secret (per-environment value, same id in test and production) or subscription secret (shared), referenced with `valueFrom.secretRef` | None. Apps read secrets at start; a changed secret needs `app restart` |
| Certificates and other files (`.pfx`, `.pem`, JSON key files) | Configuration with `type: file` and a `secretRef`; the file appears in `/app_secrets/<name>` with the name lower-cased | Read the path from configuration; do not hard-code `/app_secrets` in more than one place |
| `Litium:Data:ConnectionString`, Litium Search and Redis connection settings | Injected by the platform | Delete the transform and every `appsettings.<Env>.json` copy; leave no value in `appsettings.json` that would win over the injected one. If a `Litium:*` key looks environment-specific and is not obviously injected, keep it on the *Open questions* list |
| `Litium:Folder:Local`, `Litium:Folder:Shared` | Managed storage, set by the platform; media is restored from the storage backup artifact | Delete your values; never write outside the folders Litium gives you through its file APIs |
| Any other folder the code reads or writes (integration drops, exports, imports, report output) | A File storage app folder mounted with a `type: storage` configuration in the Litium platform manifest; the code sees it under `/app_storage/<name>` | Paths from configuration, forward slashes, no assumption that the folder already contains anything |
| Scheduler policies (`Litium:Scheduler:Policy:*`) | Keep in `appsettings.json`; override per environment with configurations if schedules differ | See section 5 for culture and time zone |

Example manifest fragment (edit the one from `litium-cloud marketplace manifest`, never author from memory; see `references/manifests.md`):

```yaml
  configurations:
    - name: LITIUM__ACCELERATOR__ERP__HOST
      value: https://erp-test.example.com
    - name: LITIUM__ACCELERATOR__ERP__PASSWORD
      valueFrom:
        secretRef:
          name: erp-password
    - name: erp_certificate.pem
      type: file
      valueFrom:
        secretRef:
          name: erp-certificate
    - name: integration
      type: storage
      subPath: integration-directory
      valueFrom:
        appRef:
          name: file-storage
          key: storage_volume
```

### Integration folders in code

Before: `var path = @"D:\Integration\Erp\Inbox";` or `Path.Combine(_configuration["Litium:Folder:Shared"], "erp", "inbox")`.

After: a configuration key (`LITIUM__ACCELERATOR__ERP__INBOXPATH=/app_storage/integration/inbox`), read with `_configuration["Litium:Accelerator:Erp:InboxPath"]`, and `Directory.CreateDirectory` before first use. The same File storage folder can be mounted into a Litium SFTP app, which is how an ERP keeps dropping files. Case matters for the folder names in `subPath`.

## 5. Runtime behavior

### Culture in background work

Web requests get their culture from the channel, as before. Scheduled jobs, Connect handlers and other background work do not: the operating system culture is not set in Serverless Cloud, so `decimal.Parse("1,5")`, `date.ToString("d")` and currency formatting silently produce different results than on the Swedish or English Windows server. A job that looks up a channel, website or language from `CultureInfo.CurrentCulture.Name`, or reads anything from a web request (`HttpContext`, the request model), gets null instead and fails with a `NullReferenceException` that never happened on Windows. Set the culture at the start of every job that formats or parses, either from the website or channel the job works for, or a fixed one:

Before:

```csharp
public Task ExecuteAsync(object parameter, CancellationToken cancellationToken)
{
    var price = decimal.Parse(row.Price);              // depended on the server culture
    var fileName = $"export-{DateTime.Now:d}.csv";
    ...
}
```

After:

```csharp
public Task ExecuteAsync(object parameter, CancellationToken cancellationToken)
{
    var culture = _websiteCulture ?? CultureInfo.GetCultureInfo("sv-SE");
    CultureInfo.CurrentCulture = culture;
    CultureInfo.CurrentUICulture = culture;

    var price = decimal.Parse(row.Price, culture);     // explicit is better even with CurrentCulture set
    var fileName = $"export-{DateTime.UtcNow.ToString("yyyy-MM-dd", CultureInfo.InvariantCulture)}.csv";
    ...
}
```

`CurrentCulture` is per thread and async-flows, so set it at the entry point of the job, not in a constructor. Prefer `CultureInfo.InvariantCulture` for file names, file contents exchanged with other systems and anything parsed by code. Apply the same to `IHostedService` implementations and Connect handlers.

### Time zone

Cron expressions use UTC unless `CronTimeZone` is set on the policy (documented default). A job that ran "at 02:00" on a server in Swedish local time now runs at 02:00 UTC unless the policy says `"CronTimeZone": "Europe/Stockholm"` (use IANA names, Windows names such as `W. Europe Standard Time` may not resolve on Linux; verify in the test environment). Code that uses `DateTime.Now` for business logic (cut-off times, "today's" orders) must use `TimeZoneInfo.ConvertTime` with an explicit zone; assume the container clock is UTC and verify it in the test environment.

### Files, temp folders and locks

- Case-sensitive paths everywhere (section 3); include configuration values and CSS/JS references.
- Temp files: use `Path.GetTempPath()`; do not assume the temp folder survives a restart or is shared between instances. Anything that must persist or be shared goes to File storage.
- File locks: Linux does not lock files on open; a job that relied on "the file is locked while the ERP writes it" must use a done-marker or rename-on-complete convention instead. Verify integration file handling with the ERP in the test environment.
- Multiple instances: production runs more than one replica, and on Litium 8.16 or later Litium can move jobs to a dedicated worker node when they need one. Code that uses a local file or a static as a lock must use Litium's distributed lock (`DisallowConcurrentDistributedExecution` on the job) instead.

## 6. What the platform provides, do not add it

| Concern | What happens in Serverless Cloud | What to remove or leave alone |
|---|---|---|
| Reverse proxy and forwarded headers | TLS terminates at Litium CDN; the platform enables forwarded-header handling so `Request.Scheme`, host and client IP are correct in the app | Remove custom `UseForwardedHeaders` setups with `KnownProxies` for the legacy load balancer. Keep code that reads `HttpContext.Connection.RemoteIpAddress` |
| HTTPS redirect and HSTS | Public traffic is HTTPS through the CDN; HSTS max age is set per domain in the back office (Settings > Globalization > Domain names), which also makes internal self-requests (order confirmation rendering) use HTTPS | Remove `UseHttpsRedirection` and `UseHsts` variants copied from templates if they loop or double-redirect behind the CDN; verify in the test environment that no redirect loop occurs and that the HSTS header is present |
| Health checks | The platform probes `/health/startup`, `/health/live` and `/health/ready`. Litium 8.8 and later answer them built-in | Litium **before 8.8**: either implement all three endpoints, or set the probe paths to `"none"` in the web project: `<LitiumCloudStartupProbePath>none</LitiumCloudStartupProbePath>`, `<LitiumCloudLiveProbePath>none</LitiumCloudLiveProbePath>`, `<LitiumCloudReadinessProbePath>none</LitiumCloudReadinessProbePath>`. Otherwise the app never becomes ready. Do not add health-check middleware that conflicts with the built-in routes on 8.8+ |
| Logging | Application logs are collected automatically to Litium Insights, and the platform disables the file targets in the NLog configuration at startup. Nothing is written to `litium.log` | Keep `nlog.config` in the artifact; keep logging through the normal logger. Remove code that reads, tails, rotates or emails log files, and any custom log-file path setting. A separate private .NET web app wires up Insights itself with the `Litium.Cloud.NLog.Extensions` package (docs: .NET web app) |
| Auto patching | Auto patching of the Litium application is always enabled (the litium-platform page; the old `auto_patch` property was removed in app version 3.1.0) | Nothing to change; do not add an `auto_patch` property to the manifest |
| Connection strings, search, Redis, storage folders | Injected (section 4) | Delete your values and transforms |
| Back office sign-in | Litium Accounts sign-in is wired by the platform | Remove Windows-credential settings |
| Session and data protection keys | Redis and shared storage are provisioned by the platform, and production runs several replicas; the docs do not state how data-protection keys and session state are shared between them | Confirm on the Litium platform page or with Litium support before go-live if the solution uses ASP.NET Core data protection or session for anything beyond Litium's defaults; verify sign-in survives a `app restart` in the test environment |
| Application restart, warm-up, app pool settings | `app restart` (rolling) and rolling deployments in production | Remove IIS warm-up modules, `applicationInitialization`, `web.config` in general |

## 7. Mail

Install an SMTP relay app (Litium's shared server, or the customer's own service through `smtp_host`, `smtp_username`, `smtp_password` properties) and point the accelerator at its exposed values instead of the legacy SMTP host. For the MVC Accelerator the keys are `Litium:Accelerator:Smtp:*`, so the manifest of the Litium platform app gets:

```yaml
  configurations:
    - name: LITIUM__ACCELERATOR__SMTP__HOST
      valueFrom: { appRef: { name: smtp, key: smtp_host } }
    - name: LITIUM__ACCELERATOR__SMTP__PORT
      valueFrom: { appRef: { name: smtp, key: smtp_port } }
    - name: LITIUM__ACCELERATOR__SMTP__ENABLESECURECOMMUNICATION
      value: "false"
    - name: LITIUM__ACCELERATOR__SMTP__USERNAME
      valueFrom: { appRef: { name: smtp, key: smtp_auth_username } }
    - name: LITIUM__ACCELERATOR__SMTP__PASSWORD
      valueFrom: { appRef: { name: smtp, key: smtp_auth_password } }
```

The exposed values are the same whether the relay uses Litium's server or the customer's, so switching later needs no code or manifest change in the sending app. Apply the sending app's manifest after the SMTP relay app is installed. Code changes: none, unless the solution bypasses the accelerator's SMTP settings with its own `SmtpClient` or a mail package; then read host, port, user name and password from the same configuration keys. The customer must add Litium's relay to the SPF record of the sender domain when using Litium's server (`references/support-requests.md` has the wording); the shared server supports neither TLS nor DKIM, so business-critical mail goes through the customer's own service. A React Accelerator storefront uses the `RUNTIME_SMTP_*` variables in its own manifest the same way (section 8).

## 8. Storefront

| Storefront | Artifact | Code change |
|---|---|---|
| MVC Accelerator | One `dotnet` artifact | Everything in this file; the client build runs before publish (section 2) |
| React Accelerator (Litium Storefront app, type `litium-nextjs-web`) | `dotnet` artifact for the platform plus a `nextjs` artifact from the `frontend` folder | No code change. `RUNTIME_LITIUM_SERVER_URL` is set automatically at install; every other `.env` value moves to `configurations` in the storefront manifest (`RUNTIME_*` at runtime; `NEXT_PUBLIC_*` and anything read in `next.config.js` are build-time and need `.env.production` plus `!.env.production` in `.litiumcloudignore`, and then one build per environment). The default health endpoints under `api/health/*` are already in the accelerator. Check `engines.node` in `package.json` and that a lock file exists, or the cloud build fails |
| Custom Next.js, Nuxt.js or Node.js storefront | `nextjs`, `nuxtjs` or `nodejs` artifact | Install as a private Next.js Web (or Nuxt.js, Node.js) app plus a Litium Storefront Proxy app in front (Next.js Web 1.0 installs the proxy automatically). Call the platform through `RUNTIME_LITIUM_SERVER_URL` on the server side, the public origin in the browser. Set probes in `package.json` (`litium-cloud.probe_*_path`, `"none"` to disable) if the app has no health routes |

## 9. Verification before the first artifact

- [ ] `dotnet publish ... --os linux -a x64` succeeds; `publish/` contains `litiumcloud.manifest.json`, `license.json`, `appsettings.json`, `nlog.config` and the built client assets, and no `appsettings.<Env>.json`
- [ ] No `win-*` runtime identifier, `.pubxml`, Web Deploy target or `Microsoft.Windows.Compatibility` left in any `.csproj`
- [ ] Every assessment hit in section 3 has a replacement or a documented reason it is safe (test-only, dead code)
- [ ] No hard-coded drive letters, UNC paths or backslash paths; every path is in configuration
- [ ] Every job, hosted service and Connect handler that formats or parses sets `CultureInfo.CurrentCulture`
- [ ] `CronTimeZone` set on schedules that must follow local time
- [ ] `Litium:Data:ConnectionString`, search, Redis and `Litium:Folder` values removed from the repo and pipeline
- [ ] Every environment-specific value listed with its manifest name and whether it is a `value`, `secretRef`, `type: file` or `type: storage`
- [ ] Litium < 8.8: probe paths set to `"none"` or endpoints implemented
- [ ] No code reads or rotates log files
- [ ] `.litiumcloudignore` present only if needed, and tested with `artifact download`
- [ ] Storefront section applied for the storefront type
- [ ] If a Linux runtime was available: the published output started and served the start page and `/Litium`

Then create the artifact from the docs and keep the id:

```bash
litium-cloud artifact create --artifact-type dotnet --file-path ./publish --name "migration test 1" --subscription <subscription-id>
```

`litium-cloud artifact show --artifact <artifact-id>` must show **Ready**; if not, read the job log (`references/troubleshooting.md`). Record the id in `MIGRATION.md`.

## 10. What NOT to change

- **Litium version.** Do not upgrade Litium in the same change as the migration unless the assessment requires it (< 8.1 must, < 8.8 and < 8.16 should be planned as their own step). One variable at a time.
- **Business logic, data model and views.** This is a hosting move. Resist refactoring while replacing Windows-only code.
- **`appsettings.json` structure.** Keep the keys; the manifest overrides them by name. Renaming keys breaks the mapping you just wrote down.
- **`nlog.config`.** Keep it; the platform adjusts it at startup. Do not delete file targets by hand or add console targets to compensate.
- **Health endpoints on Litium 8.8+.** Do not implement your own on top of the built-in ones.
- **Anything the platform injects.** Do not put connection strings, search or Redis settings, storage folders or sign-in settings into the manifest "to be safe"; a value there wins over the injected one and breaks the app.
- **Order number prefix, domains, channels, payment app configuration.** Those are environment setup and cutover decisions (`references/environment-setup.md`, `references/rehearsal-and-cutover.md`), not code.
- **The legacy pipeline.** Leave it running until the new one deploys to test (`references/pipeline-migration.md`); the legacy site stays live until go-live.
- **Secrets in the repo.** Do not "temporarily" commit a secret to make the test environment work; create the secret and reference it.
