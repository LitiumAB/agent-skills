# Litium Cloud workflows

Twelve end-to-end recipes. Every command is verified against CLI 2.10.1 and the public documentation.
Placeholders are written `<like-this>` — never invent a real id, name or domain.

All recipes assume a [context](../SKILL.md) is set:

```bash
litium-cloud context set --subscription <subscription-id> --environment <environment-id>
litium-cloud context show
```

Otherwise add `--subscription <subscription-id> --environment <environment-id>` to every command that
accepts them. Before any mutating command, confirm the target and say whether it is production
(`litium-cloud environment show` prints `Production: Yes|No`).

---

## `new-environment`

**When to use.** Standing up a brand-new environment (test, qa, prod) in an existing subscription.

**Prerequisites.** `system/contributor` (Contributor) on the subscription. CLI installed and signed in.

**Steps.**

1. Confirm who you are, and pick a location. Use the same location for test and production so they
   behave alike; an environment can never be moved afterwards.

   ```bash
   litium-cloud auth show
   litium-cloud subscription list
   litium-cloud location list
   ```

2. Create the environment. Keep the id short — it becomes part of every app's system domain
   `<subscription>-<environment>-<app>.litium.app`. Add `--production` for a production environment.

   ```bash
   litium-cloud environment create --subscription <subscription-id> --name <name> --location <location-id> --set-context
   litium-cloud environment create --subscription <subscription-id> --name prod --location <location-id> --production --set-context
   ```

3. Wait for the job, then verify.

   ```bash
   litium-cloud status show --job <job-id>
   litium-cloud environment list --subscription <subscription-id>
   litium-cloud environment show --environment <environment-id>
   ```

4. Install the apps, **in this order**: Litium CDN (`install-cdn-insights`), Litium Insights,
   Litium platform (`install-litium-platform`), then the storefront and the payment and delivery apps.

**Verify.** `environment show` prints the name, location, the production flag and an empty app list.

**Docs.** `/cloud/serverless/get-started/create-environment`, `/cloud/serverless/cli/environment`,
`/cloud/serverless/guides/configure/production-environments`.

---

## `deploy-dotnet`

**When to use.** Releasing new .NET code — a Litium platform app, an MVC accelerator solution, or a
private `dotnet-web` app.

**Prerequisites.** `subscription/reader`, `artifact/creator` on the subscription, `environment/reader`
on the environment, `appresource/writer` on the app (an Owner already has all of these). The .NET 8+
SDK for the build.

**Steps.**

1. Add the manifest generator to the startup project once — it emits `litiumcloud.manifest.json`
   (entry point, .NET version) at publish time — then publish for Linux x64. .NET artifacts are built
   **locally**, not in the cloud.

   ```bash
   dotnet add <path/to/web-project.csproj> package Litium.Cloud.Tools.Targets --prerelease
   dotnet publish <path/to/web-project.csproj> -f <target-framework> -c Release -a x64 --os linux -o publish
   ```

   Health checks default to `/health/startup`, `/health/live` and `/health/ready`. They are built in
   from Litium 8.8; on an earlier version implement them or set the probe paths to `"none"` via
   `<LitiumCloudStartupProbePath>`, `<LitiumCloudLiveProbePath>` and `<LitiumCloudReadinessProbePath>`
   in the `.csproj`.

2. Before deploying to production, capture the *artifact* property value that is running now, so you
   can roll back to it.

   ```bash
   litium-cloud app show --app <app-id> -o json
   ```

3. Upload the published folder as an artifact.

   ```bash
   litium-cloud artifact create --artifact-type dotnet --file-path ./publish/ --name "<release-name>" --subscription <subscription-id>
   ```

   In a pipeline add `--no-progress -o json` and read `data.artifactId`.

4. Wait until the artifact status is **Ready**. On `Failed`, read the build log — the artifact id
   doubles as the build job id: `litium-cloud status logs --job <artifact-id>`.

   ```bash
   litium-cloud artifact show --artifact <artifact-id>
   ```

5. Deploy and follow the job.

   ```bash
   litium-cloud app deploy --app <app-id> --artifact <artifact-id>
   litium-cloud status logs --job <job-id> --follow
   litium-cloud status show --job <job-id>
   ```

**Verify.** `litium-cloud app show --app <app-id>` shows the new *artifact* id and `State: Running`,
and the site answers on the *public_domain_name* from the **Exposes** table.

**Notes.** Deployment is a rolling update: several replicas mean no downtime, a single replica is
briefly unavailable. Deploying to a Litium platform app upgrades the database automatically when the
artifact carries a newer Litium version — **the db tool never downgrades**, so take a database backup
first (`backups`). To roll back, `app deploy` the previous artifact id; that replaces code only, not
data or a database upgrade.

**Docs.** `/cloud/serverless/guides/artifacts/create-dotnet-artifact`,
`/cloud/serverless/guides/artifacts/deploy-artifact`.

---

## `deploy-nextjs`

**When to use.** Releasing a Next.js storefront (`litium-nextjs-web`, or a private `nextjs-web` app).
The same shape applies to `nodejs` and `nuxtjs` artifacts — only `--artifact-type` changes.

**Prerequisites.** Same roles as `deploy-dotnet`. The app already installed.

**Steps.**

1. Prepare `package.json`. Next.js, Node.js and Nuxt.js artifacts are **built in the cloud from
   source**, so every dependency must be reachable — put private registry credentials in `.npmrc`.

   - Node version comes from `engines.node`; without it the newest supported version is used.
   - Health checks default to `api/health/startup`, `api/health/live`, `api/health/ready`; override or
     disable them under a `litium-cloud` key with `"none"`.
   - The package manager is chosen from the lock file, in order: `yarn.lock` (yarn 1),
     `package-lock.json` (npm), `pnpm-lock.yaml` (pnpm). **No lock file means the build fails.**

2. Check what an ignore file would strip. `.gitignore`, `.npmignore` and `.litiumcloudignore` in the
   **root** of the uploaded folder decide what is packaged, last match wins, `!` re-includes. Use
   `.litiumcloudignore` to bring back build-time env files that `.gitignore` excludes.

3. Upload the source folder, then wait for the cloud build — it takes longer than a .NET upload.

   ```bash
   litium-cloud artifact create --artifact-type nextjs --file-path ./frontend/ --name "<release-name>" --subscription <subscription-id>
   litium-cloud status logs --job <artifact-id> --follow
   litium-cloud artifact show --artifact <artifact-id>
   ```

4. Deploy and follow the job.

   ```bash
   litium-cloud app deploy --app <app-id> --artifact <artifact-id>
   litium-cloud status logs --job <job-id> --follow
   ```

**Verify.** `app show --app <app-id>` shows the new artifact and the storefront answers on its
*public_domain_name*.

**Troubleshooting.** An error page usually means the build lacked `output: 'standalone'` in
`next.config.js`, or a build-time env file was excluded from the artifact. If the app never starts,
collect its own output: `litium-cloud app action --app <app-id> --action console-output`, then read
the related job with `status logs`.

**Docs.** `/cloud/serverless/guides/artifacts/create-nextjs-artifact`,
`/cloud/serverless/apps/public-apps/litium-storefront`,
`/cloud/serverless/guides/artifacts/using-environment-variables`.

---

## `cicd-service-principal`

**When to use.** Automating deployments from Azure DevOps, GitHub Actions or any other pipeline.

**Prerequisites.** Permission to grant roles on the target resources — **Owner** or
**User access manager** (`system/acl-manager`) on the subscription or environment.

**Steps.**

1. Create the principal and its certificate. The private key is shown once and is never stored by
   Litium. Note the id from the output (`service.<name>@cloud`). A `.pfx` file name produces PKCS #12
   instead of PEM; both work for sign-in.

   ```bash
   litium-cloud service-principal create --name <principal-name> --expires 180 -f ./<principal-name>.pem
   ```

2. Store the certificate in the pipeline's secret store — an Azure DevOps secure file, a GitHub
   Actions secret — and delete the local copy. **Never commit it.** Also store the NuGet feed
   credentials the pipeline needs to install the CLI.

3. Grant the four minimum roles. A service principal inherits nothing from you.

   ```bash
   litium-cloud subscription access-control add --subscription <subscription-id> --email <service-principal-id> --role subscription/reader
   litium-cloud subscription access-control add --subscription <subscription-id> --email <service-principal-id> --role artifact/creator
   litium-cloud environment  access-control add --subscription <subscription-id> --environment <environment-id> --email <service-principal-id> --role environment/reader
   litium-cloud app          access-control add --subscription <subscription-id> --environment <environment-id> --app <app-id> --email <service-principal-id> --role appresource/writer
   ```

   Add `apps/litium-platform/backup-operator` on the app if the pipeline takes a backup first. Grant
   the narrowest scope that works — an app or an environment, not the subscription, for production.

4. In the pipeline, run these five steps after the build.

   ```bash
   dotnet tool update -g litium.cloud.cli --no-cache

   litium-cloud auth login --service-principal --username <service-principal-id> --certificate <absolute-path-to-certificate>

   litium-cloud artifact create --subscription <subscription-id> --artifact-type dotnet --file-path <publish-folder> --no-progress -o json
   # read data.artifactId

   litium-cloud artifact show --artifact <artifact-id> -o json
   # poll status: Initiated | Uploading | Processing | Ready | Failed
   # on Failed: litium-cloud status logs --job <artifact-id> and fail the build

   litium-cloud app deploy --subscription <subscription-id> --environment <environment-id> --app <app-id> --artifact <artifact-id> -o json
   # read jobId, then poll:
   litium-cloud status show --job <job-id> -o json
   # finished when completedAt is set; succeeded when no entry in items has failed = true
   ```

5. Verify the principal's access at any time.

   ```bash
   litium-cloud auth permission --email <service-principal-id>
   ```

**Verify.** The pipeline ends with the job completed and `app show --app <app-id>` reporting the new
artifact id.

**Pipeline rules.**

- Use `-o json` on every command you parse; never parse the table output.
- Sign in at the start of every run — the sign-in is cached on the agent and a stale cache surprises you.
- Pass `--no-progress` explicitly so local runs behave like CI.
- Destructive commands (`app delete`, `app pause`) abort in CI without `--auto-yes`.
- Use an **absolute** certificate path; a relative one stored in the config later fails from another
  directory with a misleading `Could not connect to server.`

**Renewal.** Certificates last 180 days by default, 365 at most. `service-principal list` shows
**Expires at**. Renewing **revokes every other active certificate**, so swap the pipeline secret in the
same maintenance window:

```bash
litium-cloud service-principal renew --service-principal <service-principal-id> --expires 180 -f ./<principal-name>.pem
litium-cloud service-principal show  --service-principal <service-principal-id>
```

**Docs.** `/cloud/serverless/guides/automated-deployments/overview`,
`/cloud/serverless/guides/automated-deployments/azure-devops`,
`/cloud/serverless/guides/automated-deployments/github-actions`,
`/cloud/serverless/guides/access/service-principals`.

---

## `install-litium-platform`

**When to use.** First installation of the Litium platform app in an environment, with or without an
existing database and media.

**Prerequisites.** Contributor on the environment. **Litium CDN installed first** — otherwise the app
has no public domain. Optionally a `dotnet` artifact of your solution (including its `license.json`),
a `sqlbackup` artifact and a `storage` artifact.

**Steps.**

1. Download the template. It carries the newest version and every property, optional ones commented.

   ```bash
   litium-cloud marketplace manifest --app litium-platform -f litium-platform.yaml
   ```

2. Edit it. Artifact references have the form `artifacts/<artifact-id>`.

   ```yaml litium-platform.yaml
   kind: app
   resource:
     id: litium
   spec:
     type: litium-platform
     version: <version>
     properties:
       - name: artifact
         value: artifacts/<artifact-id>
       - name: sql_backup_file
         value: artifacts/<sql-backup-artifact-id>
       - name: storage_backup_file
         value: artifacts/<storage-backup-artifact-id>
   ```

   | Property | Meaning |
   |---|---|
   | `artifact` | The .NET artifact to run. Omitted on create: the newest Litium version is installed. Omitted on update: the deployed artifact is kept |
   | `sql_backup_file` | Database backup restored **when the app is created**. Omitted: a fresh database in the artifact's version |
   | `storage_backup_file` | Storage backup restored when the app is created |
   | `redis_prefix` | Prefix for every Redis key. Changing it clears the cache, including live carts |

   Add environment-specific settings under `configurations`, referencing secrets with
   `valueFrom.secretRef` — never literal passwords. Keep the file in version control.

3. Apply and follow the job, then find the URL. Installation takes several minutes, longer with a
   database restore.

   ```bash
   litium-cloud apply -f litium-platform.yaml
   litium-cloud status logs --job <job-id> --follow
   litium-cloud status show --job <job-id>
   litium-cloud app show --app litium
   ```

4. Open `https://<public_domain_name>/Litium`, taking the domain from the **Exposes** table. Without a
   restored database, the first sign-in is `Admin` / `Password!` — change it immediately. With a
   restored database, use the accounts from that database.

**Verify.** `status show` reads `Status: completed` with no *failed* entry in **Details**,
`litium-cloud app list` shows the app, and the back office start page loads.

**Troubleshooting.** A failed restore usually means the backup comes from a newer Litium version than
the artifact. No response on the public domain usually means Litium CDN was missing when the app was
created — install it, then `litium-cloud app restart --app litium --wait`. A license error means the
artifact lacks a valid `license.json` for the domain.

**Docs.** `/cloud/serverless/apps/public-apps/litium-platform/overview`,
`/cloud/serverless/reference/manifest`.

---

## `install-cdn-insights`

**When to use.** Preparing a new environment. **Litium CDN must be the first app installed**; Litium
Insights is normally next.

**Prerequisites.** Contributor on the environment. One instance of each per environment.

**Steps.**

1. Litium CDN. Every app with a public endpoint is registered in it automatically when installed, and
   gets a system domain `<subscription>-<environment>-<app>.litium.app`. The Fastly image optimizer
   is active for all LCC environments.

   ```bash
   litium-cloud marketplace manifest --app litium-cdn -f litium-cdn.yaml
   litium-cloud apply -f litium-cdn.yaml
   litium-cloud status logs --job <job-id> --follow
   litium-cloud app show --app litium-cdn
   ```

   If you install the CDN after other apps, the apps installed earlier are **not** configured in the
   CDN automatically: add their system domains with the CDN's `add-domain-name` action (see
   `custom-domain`), or restart them (`litium-cloud app restart --app <app-id> --wait`) as the Litium
   platform page suggests for a platform app that does not answer on its system domain.

2. Litium Insights — application logs, request logs and metrics for every app in the environment.

   ```bash
   litium-cloud marketplace manifest --app litium-insights -f litium-insights.yaml
   litium-cloud apply -f litium-insights.yaml
   litium-cloud status logs --job <job-id> --follow
   ```

3. Grant people access to Insights. A user needs read access on the subscription plus one of the
   Insights roles: `apps/litium-insights/insights-logs` (application logs and request metrics) or
   `apps/litium-insights/insights-bi` (business intelligence). Grant the Insights role on the
   subscription to cover every Insights app in it, or on one app only.

   ```bash
   litium-cloud subscription access-control add --email <user-or-group-email> --role subscription/reader --subscription <subscription-id>
   litium-cloud subscription access-control add --email <user-or-group-email> --role apps/litium-insights/insights-logs --subscription <subscription-id>
   litium-cloud app access-control add --email <user-or-group-email> --role apps/litium-insights/insights-logs --app litium-insights --subscription <subscription-id> --environment <environment-id>
   ```

   Grant the roles to a group such as *Insights users* rather than per user — see `access-control`.

4. Sign in at https://insights.litium.cloud with the same Litium Account.

**Verify.** `litium-cloud app list` shows both apps; `app show --app litium-cdn` lists
*public_domain_name* under **Exposes**; the environment appears in Insights after signing out and in.

**Note.** Fastly access is **not** needed to install apps or to add custom domains — those are
self-service. Contact Litium support only for a certificate that is not already in Fastly, or to move
a domain between Fastly services you do not control.

**Docs.** `/cloud/serverless/apps/public-apps/litium-cdn`,
`/cloud/serverless/apps/public-apps/litium-insights`,
`/cloud/serverless/guides/operate/monitoring-with-litium-insights`.

---

## `backups`

**When to use.** Before a risky deployment or an upgrade, to copy data between environments, or to
bring data in from outside Serverless Cloud. A backup **is** an artifact.

**Prerequisites.** On Litium platform: `apps/litium-platform/backup-operator` (or Owner/Contributor).
On an MS SQL app: `apps/mssql-db/backup-operator`. On a File storage app:
`apps/file-storage/backup-operator`. To upload a backup: access to the subscription.

**Steps — database backup of a running app.** A production database takes a while; you can leave the
follow and check back later with `status show`. The artifact is named
*Database backup subscriptions/…/apps/`<app-id>`/resources/database*.

```bash
litium-cloud app action --action backup-database --app <app-id>
litium-cloud status logs --job <job-id> --follow
litium-cloud artifact list --artifact-type sqlbackup --reverse
```

**Steps — storage backup.**

```bash
litium-cloud app action --action backup-storage --app <app-id>
litium-cloud artifact list --artifact-type storage --reverse
```

By default `backup-storage` includes only what is needed to move a site between environments, in
practice the media files. For an exact copy of the whole storage folder:

```bash
litium-cloud app action --app <app-id> --action backup-storage --property complete=true
```

**Steps — download a backup.**

```bash
litium-cloud artifact download --artifact <artifact-id> -f ./<name>.bak    # sqlbackup
litium-cloud artifact download --artifact <artifact-id> -f ./<name>.zip    # storage
```

**Steps — upload a backup from outside the cloud.**

```bash
litium-cloud artifact artifact-type list
litium-cloud artifact create --artifact-type sqlbackup --file ./<name>.bak  --name "<name>" --subscription <subscription-id>
litium-cloud artifact create --artifact-type storage   --file ./files/      --name "<name>" --subscription <subscription-id>
```

For `storage`, point at the folder that contains the *media* folder; the structure inside is
preserved and ignore files in that folder's root apply. A `.bak` file is uploaded as it is.

**Verify.** `artifact list --artifact-type sqlbackup` (or `storage`) shows the artifact with
status **Ready**.

**Retention.** A backup artifact that was never used is deleted **14 days** after creation; one that
has been used is deleted **7 days** after the last app stopped using it. Download anything you must
keep, for example a pre-go-live backup.

**Note.** These actions are on-demand backups. Litium takes platform backups as part of the service
separately. To use a backup, see `restore`.

**Docs.** `/cloud/serverless/guides/backups/overview`, `…/database-backup`, `…/storage-backup`.

---

## `restore`

**When to use.** Bringing a database and media backup into an environment. Backup artifacts are
**create-only**, so a restore means **uninstall and reinstall** the app.

**Prerequisites.** Contributor on the environment. A `sqlbackup` and, if you restore media, a
`storage` artifact in the subscription. A maintenance window — the site is down from the uninstall
until the reinstall completes, typically 15 to 60 minutes depending on backup size.

**Steps.**

1. Take a fresh backup of what is there now, unless you are certain you do not need it (`backups`).

2. Find the artifact ids.

   ```bash
   litium-cloud artifact list --artifact-type sqlbackup --limit 5 --reverse
   litium-cloud artifact list --artifact-type storage   --limit 5 --reverse
   ```

3. Export the manifests of the platform app **and every app that depends on it** — storefront,
   payment and delivery apps. Keep them in source control; you need them to reinstall.

   ```bash
   litium-cloud app list
   litium-cloud app show --app <platform-app-id> -o manifest > litium-platform.yaml
   litium-cloud app show --app <dependent-app-id> -o manifest > <dependent-app-id>.yaml
   ```

4. Uninstall dependants first, then the platform app, waiting for each job.

   ```bash
   litium-cloud app delete --app <dependent-app-id>
   litium-cloud status show --job <job-id>
   litium-cloud app delete --app <platform-app-id>
   litium-cloud status show --job <job-id>
   ```

   Each prompts `Deleting an app is a destructive operation. Do you want to continue?` — answer `y`.

5. Add the backups to the exported platform manifest, keeping everything else as exported.

   ```yaml litium-platform.yaml
   kind: app
   resource:
     id: litium
   spec:
     type: litium-platform
     version: <version>
     properties:
       - name: artifact
         value: artifacts/<artifact-id>
       - name: sql_backup_file
         value: artifacts/<sql-backup-artifact-id>
       - name: storage_backup_file
         value: artifacts/<storage-backup-artifact-id>
   ```

6. Reinstall the platform app, then the dependants.

   ```bash
   litium-cloud apply -f litium-platform.yaml
   litium-cloud status logs --job <job-id> --follow
   litium-cloud apply -f <dependent-app-id>.yaml
   ```

7. Post-restore fixes.

   ```bash
   litium-cloud app action --app <platform-app-id> --action rebuild-search-indicies
   ```

   Payment and delivery apps must also be installed again inside the Litium back office. If the
   backup came from another environment, its domain names come with it — fix them with the domain
   actions (`custom-domain`).

**Verify.** `status show` reads `Status: completed` with no *failed* entry; the back office signs in
with an account from the restored database; products, media and orders are there; search works.

**Troubleshooting.** A failed database restore usually means the backup is from a newer Litium version
than the artifact. Missing media means the storage backup was taken without `complete=true` or was
not referenced. Ghost payment apps come from a legacy-cloud database — remove them in the back office
and install the Serverless Cloud versions.

**Docs.** `/cloud/serverless/guides/backups/restore-database-and-storage`.

---

## `access-control`

**When to use.** Granting, listing or removing access; setting up groups; separating production access.

**Prerequisites.** **Owner** (`system/owner`) or **User access manager** (`system/acl-manager`) on the
resource. Contributor can manage a resource but **cannot** assign roles.

**Steps.**

1. See what exists today. `--details` expands group memberships.

   ```bash
   litium-cloud subscription access-control show --details
   litium-cloud environment  access-control show --environment <environment-id>
   litium-cloud app          access-control show --app <app-id>
   litium-cloud auth permission --email <email>
   ```

2. Pick the role. `role list` is authoritative for your account; `role show` lists where it may be
   granted. Common choices: `system/reader`, `system/contributor`, `system/owner`,
   `system/acl-manager`, and the narrow `subscription/reader`, `environment/reader`,
   `appresource/writer`, `artifact/creator`.

   ```bash
   litium-cloud role list
   litium-cloud role list --filter insights
   litium-cloud role show --role <role-id>
   ```

3. Prefer a group over individual users. Note the group id from `group create` — it is in email form
   and is what you pass as `--email`. `group update --member` **replaces** the whole member list; use
   the member commands to change one person.

   ```bash
   litium-cloud group create --name "<group-name>" --description "<description>" --member <user-email> --member <other-user-email>
   litium-cloud group list
   litium-cloud group show --group <group-id>
   litium-cloud group member add    --group <group-id> --email <user-email>
   litium-cloud group member remove --group <group-id> --email <user-email>
   ```

4. Grant. `--email` takes a user email, a group id or a service principal id.

   ```bash
   litium-cloud subscription access-control add --email <email-or-group-id> --role <role-id>
   litium-cloud environment  access-control add --email <email-or-group-id> --role <role-id> --environment <environment-id>
   litium-cloud app          access-control add --email <email-or-group-id> --role <role-id> --app <app-id> --environment <environment-id>
   ```

5. Remove. Both options are required, and only an entry granted **on this resource** can be removed —
   remove an inherited entry where it was granted.

   ```bash
   litium-cloud subscription access-control remove --email <email> --role <role-id>
   ```

6. Separate production from the subscription. Disabling inheritance copies the inherited grants as
   explicit ones first, so nobody loses access at that moment.

   ```bash
   litium-cloud environment access-control disable-inheritance --environment <production-environment-id>
   litium-cloud environment access-control show --environment <production-environment-id>
   ```

   Only when you deliberately want to start from an empty list — and after granting yourself an
   explicit role — use `--convert false`; it can lock you out.

   ```bash
   litium-cloud environment access-control enable-inheritance --environment <environment-id>
   ```

**Verify.** `access-control show` lists the principal with the role. The person has to sign out and in
again for a new role or group membership to take effect.

**Note.** Some grants are protected by Litium and cannot be removed by partners. Access denied and
"not found" look the same — the platform reports a resource you may not read as missing.

**Docs.** `/cloud/serverless/guides/access/access-control`, `…/groups`,
`/cloud/serverless/reference/roles-and-permissions`.

---

## `copy-environment`

**When to use.** Replicating an environment — for example refreshing a test environment from
production, or standing up a new environment that matches an existing one.

**Prerequisites.** Access to manage both environments. A prepared, empty target environment
(`new-environment`, or **Duplicate Environment** in the Portal, which pre-fills the settings but
installs no apps).

**Steps.**

1. Inventory the source environment.

   ```bash
   litium-cloud app list --sub <subscription-id> --env <source-environment-id>
   ```

2. Export a manifest per app.

   ```bash
   litium-cloud app show --app <app-id> -o manifest --sub <subscription-id> --env <source-environment-id> > <app-id>.yaml
   ```

3. Take the data with you — a database and a storage backup of the source platform app. Wait for each
   job, then add the artifact ids to the platform manifest's `properties`.

   ```bash
   litium-cloud app action --action backup-database --app <app-id> --sub <subscription-id> --env <source-environment-id>
   litium-cloud app action --action backup-storage  --app <app-id> --sub <subscription-id> --env <source-environment-id>
   ```


   ```yaml litium-platform.yaml
   - name: sql_backup_file
     value: artifacts/<sql-backup-artifact-id>
   - name: storage_backup_file
     value: artifacts/<storage-backup-artifact-id>
   ```

4. Install into the target environment **in dependency order**, waiting for each job:
   Litium CDN → Litium Insights → Litium platform → everything else (storefront, payment, delivery,
   SMTP, private apps).

   ```bash
   litium-cloud apply -f litium-cdn.yaml       --sub <subscription-id> --env <target-environment-id>
   litium-cloud apply -f litium-insights.yaml  --sub <subscription-id> --env <target-environment-id>
   litium-cloud apply -f litium-platform.yaml  --sub <subscription-id> --env <target-environment-id>
   litium-cloud apply -f <app-id>.yaml         --sub <subscription-id> --env <target-environment-id>
   ```

5. Post-installation.

   - Recreate the secrets the manifests reference — a manifest carries the reference, not the value.
   - Get the new URLs: `litium-cloud app show --app <app-id> --sub <subscription-id> --env <target-environment-id>`.
   - Reinstall payment and delivery apps inside the Litium back office, or with the
     `uninstall-app` / `install-app` / `configure-app` actions on the platform app.
   - `litium-cloud app action --app <app-id> --action rebuild-search-indicies`.
   - Fix the domains carried over in the database with `remove-domain` and `add-domain`; add custom
     public domains with Litium CDN domain apps (`custom-domain`).

**Verify.** The target environment serves the same content and functionality as the source: apps
respond, search works, domains are correct.

**Note.** This same recipe automates a nightly refresh of a test environment from production backups.
For a routine code release, use `deploy-dotnet` / `deploy-nextjs` instead.

**Docs.** `/cloud/serverless/guides/operate/copy-environment`.

---

## `custom-domain`

**When to use.** Putting a customer domain in front of Litium platform or another app. Every app
already has a system domain `<subscription>-<environment>-<app>.litium.app`.

**Prerequisites.** Contributor on the app. Litium CDN installed. Control of the domain's DNS. A TLS
certificate for the domain **in Fastly** — see the certificate note below.

A custom domain always needs two things: the CDN must know the domain and which app it routes to,
**and** the app must accept it. Only Litium platform has the app-side `add-domain` action.

**Steps — Litium platform.**

1. Add the domain to Litium CDN by installing one Litium CDN domain app per domain. Prefer this over
   the CDN action: it shows up in `app list`, travels with the environment's manifests, and removing
   it removes the domain.

   ```yaml <domain-app-id>.yaml
   kind: app
   resource:
     id: <domain-app-id>
   spec:
     type: litium-cdn-domain
     version: <version>
     properties:
       - name: cluster_domain
         valueFrom:
           appRef:
             name: <platform-app-id>
             key: internal_domain_name
       - name: public_domain
         value: <public-domain>
   ```

   ```bash
   litium-cloud apply -f <domain-app-id>.yaml
   litium-cloud status show --job <job-id>
   ```

2. Register the domain in Litium platform, then assign it to a channel or website in the back office.

   ```bash
   litium-cloud app action --app <platform-app-id> --action add-domain --property domain=<public-domain>
   ```

3. Point DNS at the CDN with a CNAME. Litium support gives you the CNAME target for the CDN service
   and the record type for an apex domain, which cannot carry a CNAME. Lower the TTL of the records
   you will change (for example to 300 seconds) at least a day in advance, so the switch — and a
   rollback — is quick.

**Any other app** (for example a storefront) needs only the CDN side: install a Litium CDN domain app
whose `cluster_domain` references that app's `internal_domain_name`.

**Replace or remove.**

```bash
litium-cloud app action --app <platform-app-id> --action replace-domain --property domain=<old-domain> --property new_domain=<new-domain>
litium-cloud app action --app <platform-app-id> --action remove-domain  --property domain=<old-domain>
litium-cloud app delete --app <domain-app-id>
```

The equivalent CDN actions, when you are not using a domain app:

```bash
litium-cloud app action --app <cdn-app-id> --action add-domain-name    --property public_domain=<public-domain> --property cluster_domain=<internal-domain-name>
litium-cloud app action --app <cdn-app-id> --action remove-domain-name --property public_domain=<public-domain> --property cluster_domain=<internal-domain-name>
```

Read `internal_domain_name` from `litium-cloud app show --app <app-id>` under **Exposes**.

**Verify.** `https://<public-domain>` opens the site with a valid certificate.

**Certificates.** The CDN terminates TLS. If the certificate is already in Fastly (for example the
domain was served by a Litium cloud site before), the whole flow is self-service. If not, contact
Litium support **before** adding the domain and switching DNS — allow at least three working days.

**Moving a domain from a legacy cloud site.** A domain can belong to only one Fastly service at a
time. It must be removed from the old service before it is added here, and the site is unreachable in
between. Do it yourself if you have access to the legacy service, otherwise book Litium support at an
agreed time, at least three working days ahead.

**SEO.** Non-production environments and every `litium.app` system domain always answer
`noindex, nofollow`. Only a custom domain in a production environment is indexed.

**Docs.** `/cloud/serverless/guides/configure/custom-domains`,
`/cloud/serverless/apps/public-apps/litium-cdn-domain`,
`/cloud/serverless/reference/app-actions`.

---

## `app-lifecycle`

**When to use.** Restarting, pausing, resuming, re-planning or uninstalling an app, or deleting an
environment.

**Prerequisites.** Write access on the app (`appresource/writer`); Contributor on the environment to
delete apps or the environment. Get ids with `litium-cloud app list`.

**Restart.** After a configuration or secret change, after changing the production flag, or when the
app is unresponsive. Keeps the artifact, properties and data.

```bash
litium-cloud app restart --app <app-id> --wait
```

**Pause and resume.** A paused app is unreachable and its scheduled jobs and triggers do not run;
databases and storage remain (and are still billed) while the app runtime is not. Deploys to a paused
app are rejected with `App is currently paused. Resume the app before deploying.`

```bash
litium-cloud app pause  --app <app-id>
litium-cloud app resume --app <app-id>
```

`app pause` prompts `Pausing an app will make the app inaccessable. Do you want to continue?` Add
`--auto-yes` only in a deliberate script. Never pause an app handling live traffic or payment callbacks.

**Change the plan** of a private app.

```bash
litium-cloud marketplace show --app <app-type>
litium-cloud app plan --app <app-id> --plan <plan-id>
litium-cloud app plan --app <app-id> --unset
```

Changing a plan restarts the app. Public apps answer that plans are not used or cannot be modified —
those are assigned by Litium.

**Uninstall an app.** This deletes the app **and all of its data**, including databases and storage.

1. Back up first (`backups`) and export the manifest so you can reinstall:

   ```bash
   litium-cloud app show --app <app-id> -o manifest > <app-id>.yaml
   ```

2. Delete dependants first — the storefront and the payment and delivery apps before the platform.

   ```bash
   litium-cloud app delete --app <app-id>
   litium-cloud status show --job <job-id>
   ```

   `action: delete` in a manifest does **not** work for `kind: app` — use this command.

**Delete an environment.** Removes every app, database, storage, domain and secret in it.

```bash
litium-cloud environment delete --subscription <subscription-id> --environment <environment-id>
```

**This command has no confirmation prompt.** Read the ids back to the user, take backups and export
the app manifests (`copy-environment`) before running it.

**Follow every one of these.** They all start jobs:

```bash
litium-cloud status logs --job <job-id> --follow
litium-cloud status show --job <job-id>
litium-cloud app job --app <app-id> --limit 10 --reverse
litium-cloud query jobs --completed --cancelled --interval 'last 3 days'
```

**Verify.** `app list` / `environment list` reflect the change once the job completes.

**Docs.** `/cloud/serverless/guides/operate/manage-app-lifecycle`,
`/cloud/serverless/guides/operate/jobs-status-and-logs`, `/cloud/serverless/cli/app`.
