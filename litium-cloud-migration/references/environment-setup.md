# Environment setup

Phase 4 (test environment) and the first half of phase 6 (production environment). Sources of truth:
https://docs.litium.dev/cloud/serverless/migration/set-up-test-environment.md and
https://docs.litium.dev/cloud/serverless/migration/prepare-production.md. Command syntax comes from the
`litium-cloud-cli` skill; manifest content from `references/manifests.md`; code prerequisites from
`references/code-changes.md`.

Build **test first**, verify it with the customer, then build production from the same manifest files.
Nothing here is a one-off: everything that fails is fixed in code or in a manifest, never by hand in the
environment, or the production build will not reproduce it.

## Target topology

```text
                                  internet
                                     |
                        +------------v-------------+
                        |        Litium CDN        |  litium-cdn, one per environment
                        |    (Fastly, TLS ends)    |  + one litium-cdn-domain app per custom domain
                        +---+------------------+---+
                            |                  |
             +--------------v------+   +-------v----------------------+
             |   Litium platform   |<--|   Litium Storefront          |  litium-nextjs-web
             |   litium-platform   |   |   (headless sites only)      |  (nextjs artifact)
             |   (dotnet artifact) |   +------------------------------+
             +--+---+---+---+---+--+
                |   |   |   |   +----> payment / delivery apps, one per provider, version pinned
                |   |   |   +--------> SMTP relay  litium-smtp  --> customer SMTP or Litium mail server
                |   |   +------------> File storage  file-storage  <-- Litium sFTP  litium-sftp, one per user
                |   +----------------> Litium Insights  litium-insights  (app logs, requests, BI)
                +--------------------> SQL + search index + Redis + media storage (provisioned by the app)
```

The Litium platform app provisions its own database, search index, Redis cache and media storage; you never
create or connect those. Everything else in the diagram is an app you install from a manifest.

## Rules for every step

- **Manifests in version control.** One file per app under `deploy/serverless/<env>/` in the customer repo
  (`deploy/serverless/test/`, `deploy/serverless/production/`). The two folders hold the *same* file names
  and the same content; only artifact ids and the values behind `secretRef` differ. Commit them with the code.
- **Never author a manifest from memory.** Start from `litium-cloud marketplace manifest --app <type> -f <file>`,
  or `litium-cloud app show --app <id> -o manifest` for something already installed. See `references/manifests.md`.
- **Follow every job.** `apply` prints a job id; run `litium-cloud status logs --job <job-id> --follow`, then
  `litium-cloud status show --job <job-id>` and read **Details**. A parent job can read *completed* while a
  child item is *failed*.
- **Production actions**: run `litium-cloud context show` and `litium-cloud environment show` first and state
  the subscription, environment and production flag before anything that changes production.

## Install order

| # | Step | App type | `litium-cloud-cli` recipe | Why here |
|---|------|----------|---------------------------|----------|
| 1 | Environment | — | `new-environment` | Everything is scoped to it |
| 2 | Litium CDN | `litium-cdn` | `install-cdn-insights` | Apps installed afterwards get their system domain configured automatically |
| 3 | Litium Insights | `litium-insights` | `install-cdn-insights` | Installed before the platform so the first startup logs are already collected (the docs get-started guide installs it after the platform; both orders work) |
| 4 | Litium platform | `litium-platform` | `install-litium-platform`, `restore` | Restores the legacy database and media; every dependent app needs it |
| 5 | Storefront | `litium-nextjs-web` | `deploy-nextjs` | Connects to the platform automatically once both exist |
| 6 | Payment and delivery apps | one per provider | `app-lifecycle` | Legacy registrations must be force-deleted from the restored database first |
| 7 | File storage + Litium sFTP | `file-storage`, `litium-sftp` | `app-lifecycle` | sFTP mounts folders of File storage |
| 8 | SMTP relay | `litium-smtp` | `app-lifecycle` | Platform and storefront reference its exposed values |
| 9 | Litium CDN domain apps | `litium-cdn-domain` | `custom-domain` | Needs the target app's `internal_domain_name` |

Steps 7 and 8 add `appRef` configurations to the Litium platform manifest, so the platform manifest is applied
again after them. If the mounts and SMTP keys are already known from the assessment, install File storage and
the SMTP relay **before** step 4 and put the `appRef` configurations in the first platform apply; that saves an
update job and a restart. Re-applying the platform manifest later is safe: `sql_backup_file` and
`storage_backup_file` are create-only and ignored on an existing app.

## Step 1 — Environment

- **Decide**: environment id and name (`test`, `production`), location (`litium-cloud location list`), and the
  production flag. Test gets **no** `--production`; production gets it at creation, because setting it later
  requires a restart or redeploy of the platform and storefront apps.
- **After**: set the context so later commands do not need `--subscription`/`--environment`. In production,
  restrict access: grant the production environment explicitly instead of inheriting from the subscription, and
  give the pipeline's service principal only the deployment roles (recipe `access-control`,
  `references/pipeline-migration.md`).

## Step 2 — Litium CDN

- **Decide**: nothing; the app has no properties or configurations.
- **After**: the environment has its own Fastly service. Ask Litium support for access to it if the partner will
  move domains at go-live (`references/support-requests.md`). If Litium CDN is installed *after* other apps,
  those apps' system domains are not registered in the CDN; use the CDN's `add-domain-name` action to add them.

## Step 3 — Litium Insights

- **Decide**: who needs access to it, and that logs go here rather than to files on disk.
- **After**: sign in at insights.litium.cloud with the same Litium Account and open **Analytics > Dashboard >
  App Logs** for the environment. This is where you confirm the platform app started in step 4.

## Step 4 — Litium platform

The migration step. Three artifacts go into one install.

- **Decide**: which `dotnet` artifact (the first Linux build from phase 3, or the pipeline's build later),
  which database backup and which files backup, and every configuration value moved out of `appsettings.*`.
- **Manifest properties that matter**: `artifact` (`artifacts/<dotnet-artifact-id>`), `sql_backup_file`
  (`artifacts/<sqlbackup-artifact-id>`), `storage_backup_file` (`artifacts/<storage-artifact-id>`), and
  `configurations` with `secretRef` for every secret. All three artifact properties are strings of the form
  `artifacts/<id>`; the two backup ones are **create-only**.
- **Before applying**: create every secret the manifest references (recipe `access-control` for the roles,
  `install-litium-platform` for the flow); `litium-cloud environment secret list` must list all of them.
  Upload the two backups and wait until `litium-cloud artifact show` reports **Ready**. Check the files backup
  has no `IdentityServer` folder and that `--file-path` points at the folder that *contains* `media`.
- **After**: `litium-cloud app show --app litium` shows `public_domain_name` under **Exposes**; open it with
  `/Litium` appended and sign in with a legacy administrator account. If search returns nothing, run the
  `rebuild-search-indicies` action. If the app never becomes ready, run the `console-output` action
  (`references/troubleshooting.md`).
- To restore a *different* backup later, the app is uninstalled and installed again — recipe `restore`.

## Step 5 — Storefront

- **Decide**: Litium Storefront (`litium-nextjs-web`, React Accelerator) or a private storefront app for a
  custom front end. Custom storefronts also need the storefront proxy; confirm the setup with Litium support.
- **Manifest properties that matter**: `artifact` pointing at a `nextjs` artifact. Configurations are the
  storefront's own environment variables, including the SMTP ones if it sends mail.
- **After**: add the storefront's public domain to a channel in the back office, then check that pages render
  from the migrated database.

## Step 6 — Payment and delivery apps

- **First, force-delete the legacy registrations.** The restored database still lists the apps that were
  installed in the legacy site. A normal uninstall fails, so force-delete each one — in the back office (apps
  page under **Settings**, **Uninstall**, then the force-delete option in the dialog when it fails), or with the
  `uninstall-app` action on the platform app with `id=<legacy-app-id>` and `force=true`, which needs the
  `apps/litium-platform/litium-management` role.
- **Decide**: which apps and **which version** — always pin `version` in the manifest, or the default version
  moves under you. Find the type ids with `litium-cloud marketplace list --filter payment` and
  `--filter shipment`.
- **After each app**: install it *into Litium* as well. Either open the app's `public_domain_name` and select
  **Install**, or run the `install-app` action on the platform app with `url=<the app's public URL>`. Then
  upload the provider's configuration file for this environment (back office, or the `configure-app` action).
  Record every callback URL for the webhook inventory.

## Step 7 — File storage and Litium sFTP

- **Requires the App Cloud agreement.** If the customer does not have it, stop and say so; do not design around it.
- **Decide**: which folders the integrations read and write (from the assessment), and one `litium-sftp` app
  **per external user** — separate credentials, separate folders, and at most five IP addresses per app.
- **Manifest properties that matter**: on `litium-sftp`, the `ip` property (comma- or newline-separated allow
  list) and one `type: storage` configuration per folder with `subPath` and an `appRef` to the File storage
  app's `storage_volume`. The same configuration shape mounts the folder in the platform app, where the code
  reads it under `/app_storage/<name>`.
- **After**: `litium-cloud app show --app <sftp-app-id>` lists `sftp_hostname` and `sftp_username` under
  **Exposes**; the password is a secret reference, read with `litium-cloud environment secret value`. Hand the
  host, user and password to the external system's owner over a secure channel — never in `MIGRATION.md`.
  Apply the platform manifest again so the same folders are mounted there.

## Step 8 — SMTP relay

- **Decide**: the customer's own SMTP service (set `smtp_host`, `smtp_username`, `smtp_password`) or the shared
  Litium mail server (set nothing). The shared server is for transactional mail only — no TLS, no DKIM, shared
  reputation — so anything business critical relays through the customer's own service.
- **After**: add the SMTP configurations to the platform manifest (and the storefront manifest) referencing the
  relay's exposed `smtp_host`, `smtp_port`, `smtp_auth_username`, `smtp_auth_password`, and apply. If the shared
  Litium server is used, the customer adds `include:mail3.litiumdrift.se` to the SPF record of the sending
  domain. Verify with one real transactional mail.

## Step 9 — Domains

- **Test environment**: one test domain such as `test.<customer-domain>`. Install a `litium-cdn-domain` app with
  `cluster_domain` referencing the platform app's `internal_domain_name`, run the `add-domain` action on the
  platform app, map the domain to a channel in the back office, and point DNS at the CDN. If no certificate in
  Fastly covers it, ask support (`references/support-requests.md`). Non-production environments always answer
  `noindex, nofollow`, so the test site cannot end up in search engines.
- **Production**: the same two halves (CDN app + `add-domain`), but the switch itself is planned in
  `references/rehearsal-and-cutover.md`. Use `replace-domain` when a channel must move from the test domain to
  the live one. A `litium-cdn-domain` app cannot be updated: to change the domain or its target, delete it and
  install it again.

## Verification before you call the environment done

Work through the docs' list with the customer: site on the system domain and the test domain on every channel;
back office sign-in with a legacy administrator account; search results; media; a test order through checkout,
payment and confirmation with the callback updating the order; delivery options; every integration and
scheduled job with correctly formatted numbers, dates and currencies; mail delivered; sFTP users connected;
no *File not found* or path errors in Insights; no configuration value still pointing at a legacy server.

## Production environment: what differs from test

Same manifests, different values. Create it with `--production` (worker node on 8.16+, production-sized plans,
custom domains indexable). Restrict access and grant the service principal only the deployment roles. Create
every production secret with the same ids as in test. Install Litium CDN and Litium Insights from the test
manifests. The platform app itself is not installed until the rehearsal, and then again at go-live from the
final backups.

## Record in MIGRATION.md

Environment ids and their production flag, the app id of every installed app, the artifact ids used, the secret
ids created (never values), the sFTP hosts and users (not passwords), the test domain, and the verification
list with a pass or fail per item.
