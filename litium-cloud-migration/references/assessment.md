# Assessment

Run this before anything is created in Serverless Cloud. The output is (1) the answered questionnaire, (2) the repo inventory, (3) the red-flag list, and (4) a `MIGRATION.md` in the customer repo created from `assets/MIGRATION.md`. Everything later depends on it, and it is what the person running go-live works from.

Source of truth for the inventory items: https://docs.litium.dev/cloud/serverless/migration/prepare.md ("Assess the solution").

## 1. Customer questionnaire

Ask these; they cannot be read from the repo. Record the answers under *Inventory* and *Decisions* in `MIGRATION.md`.

### Agreements and access

- Does the customer have an **LCC** (Litium Commerce Cloud) agreement? (required)
- Does the customer have the **App Cloud** agreement? (required for File storage, Litium SFTP users' folders, private .NET or Node apps, an own MS SQL database)
- Does a Serverless Cloud **subscription** exist for the customer, and which Litium Accounts already have access? (subscriptions are created by Litium; ask support for access, see `references/support-requests.md`)
- Who at the partner and at the customer owns the migration, and who approves the change freeze?

### Domains and CDN

- Is the site behind **Fastly through Litium today** (LCC)? This decides the domain path: Fastly move (no DNS change, no new certificates) or DNS switch (certificates via support, TTL lowered a day before).
- Every domain and subdomain in use, including redirect-only domains and the back office domain.
- Where DNS is managed, who can change it, and the current TTLs.
- Should the partner get **Fastly access** to manage domains themselves, or will support move the domains?

### Payments, deliveries and webhooks

- Which payment and delivery apps are installed in the back office, which versions, and where their configuration files are.
- How long after an order can the payment provider still call back (captures, refunds, cancellations)? This sets how long the old webhook URL mapping must stay in place.
- Any other inbound URLs external systems call (ERP, PIM, delivery notifications).

### Integrations, SFTP and mail

- Which external systems use **SFTP**: user names, their public IP addresses (at most five per Litium SFTP app), folders read and written.
- Which SMTP server sends mail today, and whether the customer will use their own SMTP service or Litium's relay; who updates SPF/DKIM for the new sender.
- Which scheduled jobs and integrations exist that the repo does not show (agents or scripts on the legacy servers, external schedulers).

### Go-live decisions

- Go-live date and time window (low traffic), and the code freeze date.
- Stop sales during the cutover (checkout toggle with a maintenance message) or let sales continue and import the legacy orders afterwards?
- If sales continue: new order number prefix in the new site?
- What happens to back office changes made after the final database backup (they must be redone).
- Rough size of media storage and database (drives whether the storage upload is a T-1 step).

## 2. Repo inventory (do this yourself)

Run from the solution root. Adjust paths if the solution does not follow the accelerator layout (`Src/Litium.Accelerator.Mvc` etc.). Record findings, not raw output.

### Litium version and target framework

```bash
grep -rn --include=*.csproj --include=Directory.Packages.props -E 'PackageReference Include="Litium\.' . | grep -oE 'Litium\.[A-Za-z.]+"[^/]*Version="[^"]+"' | sort -u
grep -rn --include=*.csproj -E '<TargetFramework|<RuntimeIdentifier|<PlatformTarget' .
```

- Record the highest `Litium.*` version as the platform version; confirm it against the database version the customer reports.
- `RuntimeIdentifier` set to `win-*` must go; the publish command targets `--os linux -a x64`.

### Storefront type

```bash
ls Src | grep -iE 'Mvc|Web|Storefront'; ls package.json Src/*/package.json 2>/dev/null
grep -rln --include=package.json -E '"next"|"nuxt"' . | grep -v node_modules
grep -rn --include=*.csproj -E 'Litium\.Accelerator\.Mvc|Litium\.Web' . | head
```

MVC Accelerator → one `dotnet` artifact. React Accelerator / Next.js → a `dotnet` artifact for the platform plus a `nextjs` artifact for the storefront app. Custom storefront → a private storefront app.

### Windows-only dependencies

```bash
grep -rn --include=*.csproj -E 'System\.Drawing\.Common|Microsoft\.Web\.Administration|System\.DirectoryServices|Microsoft\.Windows\.Compatibility|System\.Management|System\.ServiceProcess|Microsoft\.Win32\.Registry' .
grep -rn --include=*.cs -E 'System\.Drawing|Registry\.(Local|Current)|DirectoryServices|Microsoft\.Web\.Administration|\[DllImport|ServiceController|EventLog\.' . | grep -v /obj/ | head -50
grep -rn --include=*.csproj --include=*.pubxml --include=*.targets -iE 'MSDeploy|WebDeploy|PublishProfile|MsDeployServiceUrl|DeployIisAppPath' .
```

Each hit is a code change (`references/code-changes.md`). `System.Drawing` is the usual one (image processing, barcodes, PDF). Web Deploy targets and `.pubxml` profiles are removed with the pipeline.

### Paths and file access

```bash
grep -rn --include=*.json --include=*.config -E '"Folder"|Litium:Folder|Local"|Shared"' . | grep -v node_modules
grep -rn --include=*.cs --include=*.json --include=*.config -E '[A-Za-z]:\\\\|\\\\\\\\[a-zA-Z0-9]|Path\.Combine\("[A-Z]:' . | grep -v /obj/ | head -30
grep -rn --include=*.cs -E 'File\.(Read|Write|Open|Exists)|Directory\.(GetFiles|Exists|Create)|StreamWriter\(|StreamReader\(' . | grep -v /obj/ | grep -v /Tests/ | head -50
```

- `Litium:Folder:Local` and `Litium:Folder:Shared` are managed by the platform in Serverless Cloud; media is restored from the storage artifact. Any *other* folder the code reads or writes (integration drops, exports, temp files that must persist) becomes a File storage mount under `/app_storage/<name>`.
- Absolute Windows paths, backslashes and UNC paths must go. File and folder names must match case exactly.

### Configuration files and transforms

```bash
find . -name 'appsettings*.json' -not -path '*/node_modules/*' -not -path '*/bin/*' -not -path '*/obj/*'
find . -iname '*.transform' -o -iname 'web.*.config' -o -iname 'Transformation*.txt' | grep -v node_modules
grep -rn -E 'ASPNETCORE_ENVIRONMENT|Litium:Data:ConnectionString|Litium/Data/ConnectionString|Elasticsearch|Redis' --include=*.json --include=*.yml --include=*.yaml --include=*.txt --include=*.ps1 . | grep -v node_modules | head -40
```

- Only `appsettings.json` and `appsettings.production.json` are loaded in Serverless Cloud. Every `appsettings.<Env>.json`, Magic Chunk / XDT transform and pipeline variable becomes a manifest configuration (`ENV__VAR` naming) or a secret (`secretRef`).
- SQL, Elasticsearch and Redis connection settings are injected by the platform; remove the legacy transform that sets `Litium:Data:ConnectionString`. If a `Litium:*` key looks environment-specific and is not obviously injected, keep it on the list and confirm with Litium support.
- Note every value that differs between test and production: those become environment secrets with the same id in both environments.

### web.config

```bash
find . -name web.config -not -path '*/node_modules/*' -not -path '*/bin/*' | xargs -I{} sh -c 'echo "== {}"; grep -nE "<rewrite|<rule |maxAllowedContentLength|maxRequestLength|requestTimeout|<httpProtocol|customHeaders|<security|hstsMaxAge|aspNetCore " {}'
```

There is no IIS in Serverless Cloud; `web.config` is not read. Map each item:

| web.config item | Serverless Cloud equivalent |
|---|---|
| `<rewrite>` rules (www redirect, https redirect, trailing slash, legacy URLs) | ASP.NET Core rewrite/redirect middleware in the app, or a Litium redirect in the back office; rules in the CDN itself need access to the environment's Fastly service (ask support for access, see the Litium CDN page) |
| `maxAllowedContentLength` / `maxRequestLength` | Request bodies are capped at 100 MB and not configurable per environment; chunk larger uploads in the app |
| `requestTimeout`, `processPath`, `hostingModel`, `stdoutLogEnabled` | Not applicable; the platform runs the app directly |
| `customHeaders`, HSTS, security headers | Middleware in the app; verify behind the CDN that HTTPS redirects do not loop (TLS terminates at Litium CDN) |
| IP restrictions, basic auth | Not available in `web.config`; use app code or a Litium SFTP allow list for file transfer |

### Scheduled jobs and culture

```bash
grep -rn --include=*.json -iE 'ScheduledTask|Scheduler|"Cron"|Schedule' . | grep -v node_modules | head
grep -rln --include=*.cs -E 'ScheduledTask|IScheduledTask|Cron|BackgroundService|IHostedService' . | grep -v /obj/
grep -rn --include=*.cs -E 'CultureInfo\.(CurrentCulture|CurrentUICulture|InvariantCulture)|\.ToString\("[a-zA-Z]"\)|decimal\.Parse|DateTime\.Parse|double\.Parse' . | grep -v /obj/ | grep -v /Tests/ | wc -l
```

The exact job registration differs between Litium versions; find where the solution lists its jobs (appsettings scheduler section and classes implementing the scheduled task interface). For every job that formats or parses numbers, dates or currencies, record that it needs `CultureInfo.CurrentCulture` set explicitly; web requests are unaffected. On Litium 8.16+ in production, Litium can move jobs to a dedicated worker node if they compete with web traffic.

### SMTP

```bash
grep -rn --include=*.json --include=*.cs -iE 'Smtp|MailKit|SmtpClient|SendGrid' . | grep -v node_modules | grep -v /obj/ | head
```

Record the configuration keys (for the MVC Accelerator typically `Litium:Accelerator:Smtp:*`); they become manifest configurations referencing the SMTP relay app's exposed values (`smtp_host`, `smtp_port`, `smtp_auth_username`, `smtp_auth_password`).

### HSTS, HTTPS redirect and forwarded headers

```bash
grep -rn --include=*.cs -E 'UseHsts|UseHttpsRedirection|UseForwardedHeaders|RequireHttps|AddHsts' . | grep -v /obj/
```

Record each; verify in the test environment that the app works behind the CDN without redirect loops.

### License

```bash
find . -name license.json -not -path '*/node_modules/*'; grep -rn -i 'license.json' --include=*.yml --include=*.yaml --include=*.ps1 --include=*.csproj . | head
```

`license.json` must be inside the artifact. Record where it comes from today (repo, secure file in the pipeline, placed on the server).

### Existing pipeline

```bash
find . -maxdepth 3 \( -name 'azure-pipelines*.yml' -o -path '*/.github/workflows/*.yml' -o -name '*.ps1' -o -name '*.pubxml' -o -name '*.publishsettings' \) -not -path '*/node_modules/*'
grep -rn -iE 'msdeploy|PublishServer|publishsettings|WinSCP|sftp|Octopus' --include=*.yml --include=*.yaml --include=*.ps1 . | grep -v node_modules | head
```

Record: build steps (yarn builds, `dotnet publish` arguments), where the license and config transforms happen, the release mechanism (msdeploy.exe with `recycleApp`, SFTP upload, install agent), and every pipeline variable. `references/pipeline-migration.md` maps each to the new pipeline.

### Serverless Cloud side

With the CLI signed in (`litium-cloud-cli`): note the subscription id, existing environments and their production flag, installed apps, and the current default versions of `litium-platform`, `litium-cdn`, `litium-insights` and the payment/delivery apps (`marketplace list --details`). Note the artifact type ids from `artifact artifact-type list` (the docs use `dotnet`, `sqlbackup`, `storage`, `nextjs`).

## 3. Red flags that change the plan

| Finding | Consequence |
|---|---|
| Litium **< 8.1** | Stop. Upgrade to 8.1+ first (`litium-developer`); the migration pages do not apply |
| Litium **< 8.8** | No built-in health endpoints: set the probe paths to `"none"` or implement `/health/startup`, `/health/live`, `/health/ready` before the first artifact, or the app never becomes ready |
| Litium **< 8.16** | A dedicated worker node is not available, so jobs always share capacity with web requests. From 8.16 in production, Litium can activate one at no extra cost when jobs need it. Recommend upgrading before or soon after migration; plan load accordingly |
| SFTP, integration folders, private services or an own database needed, but **no App Cloud agreement** | Cannot install File storage or private apps. Customer must sign the agreement before the test environment; do not design around it |
| **Not on Fastly today** | Certificates for every domain come from support (lead time), DNS records change at go-live, TTL must be lowered to 300 s a day before, and the domain switch cannot be rehearsed with real traffic |
| **Many custom domains** (more than a handful, or several redirect domains) | One Litium CDN domain app per domain, a certificate check per domain, and a longer domain step on T-0; have support confirm certificate coverage for the full list |
| **Large media storage** (tens of GB) | The storage upload is the longest step; it is a T-1 step, and the customer must stop media uploads before the backup is taken. Ask support whether they upload the artifact directly |
| Production database **older than the code's Litium version** | The install upgrades it automatically; measure this in the rehearsal, and confirm the upgrade path with `litium-developer` if it spans several minor versions |
| **Windows-only packages** in core flows (image processing, PDF, barcodes) | Replacement is a real development task; schedule it before the test environment, not after |
| Code that **reads or rotates log files** | File targets are disabled at startup; remove the dependency, logs go to Litium Insights |
| **Sales continue** during the window | Order import from the legacy database after go-live is manual work; agree the order prefix and who does the import before setting the date |
| Pipeline **has no Linux publish** and uses config transforms | The pipeline is rewritten, not patched; budget time for `references/pipeline-migration.md` |

Any red flag goes into *Open questions* or *Decisions* in `MIGRATION.md`, and you tell the user before moving on.

## 4. Write MIGRATION.md

Copy `assets/MIGRATION.md` to the root of the customer repo if it is not there. Fill in:

- **Status**: phase = Assess, date, who ran it.
- **Inventory**: one line per item in the docs' inventory table (version, storefront type, integrations and jobs, SFTP, mail, payment/delivery apps, webhooks, domains, storage paths, Windows-only dependencies, configuration, license, pipeline), each with the evidence (file path or command).
- **Red flags**: the table rows that apply, with the consequence.
- **Decisions**: dates, transaction strategy, order prefix, domain path (LCC or not), who moves domains, who owns DNS.
- **Support requests**: what has been sent to support and when, what was received (artifact ids, access confirmations).
- **Open questions**: everything you had to write "confirm with Litium support" or "ask the customer" for.

Never write secret values, connection strings or certificate contents into it. Subscription, environment, app and artifact ids are fine. Commit it with the code changes so the next session, or the next person, starts from it.
