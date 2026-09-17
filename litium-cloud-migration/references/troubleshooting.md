# Troubleshooting

Symptom, cause, fix, verify. Seeded from the SKILL.md common-mistakes table, the migration pages and
https://docs.litium.dev/cloud/serverless/faq.md. Command syntax: `litium-cloud-cli`. Where a fix is a code
change, `references/code-changes.md` has the detail.

**Read failures in this order:** `litium-cloud status show --job <job-id>` → find the first item in **Details**
with status *failed* → `litium-cloud status logs --job <child-job-id>` → read the **first** error, not the last.
Job states are *pending*, *in progress*, *succeeded*, *failed*, *cancelled*. If you lost the job id,
`litium-cloud app job --app <app-id> --limit 10`.

## Startup and deployment

**A job says completed but nothing works.**
Cause: a parent job reports *completed* as soon as it finishes, whether or not the work succeeded; a child item
underneath it is *failed*.
Fix: never judge by the parent. Read **Details** in `status show` and run `status logs` on the failed child.
Verify: every item in **Details** reads *succeeded*.

**The app never becomes ready after an install or a deploy.**
Cause: the app crashes during startup, or it does not answer the health probes, so traffic is never routed to it
and nothing reaches Litium Insights.
Fix: run the `console-output` action on the app, then read the log of the *related* console-output job:
`litium-cloud app action --action console-output --app <app-id>`, then `litium-cloud status show --job <job-id>`
to find the related job id, then `litium-cloud status logs --job <related-job-id>`. The usual causes are a
missing `license.json`, a configuration value that is missing or still points at a legacy server, and
`ASPNETCORE_ENVIRONMENT` set to `Development`.
Verify: `litium-cloud app show --app <app-id>` reports **Running** and the system domain answers.

**Litium below 8.8 never becomes ready, with no error in the console output.**
Cause: built-in health endpoints arrived in Litium 8.8; earlier versions never answer the probes.
Fix: set the probe paths to `"none"` in the startup project, or implement `/health/startup`, `/health/live` and
`/health/ready`. Rebuild the artifact.
Verify: the deploy job completes and the app reports **Running**.

**Build or startup fails with `PlatformNotSupportedException` or a missing native library.**
Cause: a Windows-only package is still referenced (`System.Drawing.Common`, `Microsoft.Web.Administration`,
`System.DirectoryServices*`, `System.Management`, ...).
Fix: replace it, rebuild the artifact, deploy again.
Verify: the app starts, and the code path that used the package works in the test environment.

**Litium Insights shows nothing for the app.**
Cause: the app never started, so no application logs exist yet.
Fix: use `console-output` as above. Insights only has logs from a running app.
Verify: startup lines appear under **Analytics > Dashboard > App Logs** for the environment.

**Background jobs compete with web requests in production.**
Cause: background jobs run in the web app by default. A dedicated worker node is not part of every production
environment — Litium activates one when it is needed, at no extra cost, and it requires Litium 8.16 or later
*and* the production flag.
Fix: contact Litium support, describe which jobs slow the site down and when, and give the subscription id,
environment id and Litium version. Check `litium-cloud environment show` for the production flag first; if the
flag was set after the apps were installed, restart or redeploy the platform and storefront apps. There is no
manifest property, app action or Portal setting that turns a worker node on.
Verify: `environment show` reads production, and response times during job runs no longer rise in Insights.

## Artifacts and backups

**An artifact stays in `Processing` for a long time, or the upload crawls.**
Cause: `nextjs`, `nodejs` and `nuxtjs` artifacts are built after upload, so minutes in `Processing` are normal
(a `dotnet` artifact is built locally before the upload and is only validated afterwards) — but a huge package
usually means `node_modules`, `.git`, test output or media went along.
Fix: point `--file-path` at the published output, not the source folder; check the ignore rules; add
`--no-progress` in a pipeline. In a polling loop, raise the retry count rather than the sleep interval.
Verify: `litium-cloud artifact show --artifact <artifact-id>` reads **Ready**.

**The artifact status is `Failed`.**
Cause: the build step after the upload failed.
Fix: the artifact id doubles as its build job id — `litium-cloud status logs --job <artifact-id>`.
Verify: a rebuilt artifact reaches **Ready**.

**`apply` succeeds but the database and media are not restored.**
Cause: `sql_backup_file` and `storage_backup_file` are **create-only**. On an app that already exists they are
ignored, so the manifest looks right and nothing happened.
Fix: uninstall the dependent apps and then the platform app (show each `app delete` and get an explicit yes),
then apply the manifest again so the app is *created* with the backup ids (`litium-cloud-cli` recipe `restore`).
Verify: the back office shows the restored orders and content; `app show` lists the expected artifact.

**The storage artifact is gone on go-live day.**
Cause: it was uploaded far ahead and never referenced; an artifact that no app references is deleted 14 days
after creation if it was never used, or 7 days after the last app stopped using it (artifacts overview, FAQ).
Fix: upload on T-1, confirm **Ready**, note the id, and download a copy of anything you must keep.
Verify: `litium-cloud artifact show` still resolves the id on the morning of go-live.

## Configuration and files

**`File not found` for a file that is in the artifact.**
Cause: Linux file systems are case sensitive; a path that worked on Windows does not.
Fix: match folder and file names character for character in code and configuration, including the extension.
Rebuild and deploy.
Verify: the code path runs, and Insights shows no path errors.

**A file that must be in the artifact is missing.**
Cause: an ignore rule excluded it. The CLI starts from everything under the uploaded folder, always drops
`.git/**`, then applies the lines of `.gitignore`, `.npmignore` and `.litiumcloudignore` — read from the root
of that folder only — as one ordered list where the **last** matching rule wins. A `.gitignore` in a subfolder
has no effect, and nothing is applied to a zip file passed to `--file-path`.
Fix: re-include it with `!<pattern>` in a root `.litiumcloudignore`. Inspect the package with
`litium-cloud artifact download`.
Verify: the file is present in the downloaded artifact and the app finds it.

**Values from `appsettings.Staging.json` (or any other environment file) are missing.**
Cause: only `appsettings.json` and `appsettings.production.json` are loaded, in every environment.
Fix: move the values into manifest `configurations` (`ENV__VAR` naming) and secrets referenced with `secretRef`. Do
not set `ASPNETCORE_ENVIRONMENT` to make the file load; `Development` stops the app from starting.
Verify: `litium-cloud app show --app <app-id>` lists the configurations, and the feature behaves in test.

**Wrong number, date or currency formats in a scheduled job's output, or a `NullReferenceException` in a job that
worked on Windows.**
Cause: the operating system culture is not set in Serverless Cloud. Web requests are unaffected because Litium
sets the culture from the channel, but background work is not. On the legacy Windows server the job inherited the
server's regional setting, so a channel, website, language or format looked up from `CultureInfo.CurrentCulture.Name`
resolved; in Serverless Cloud that lookup returns null and the next line throws. Anything read from a web request
(`HttpContext`, the accelerator's request model) is null in a job as well.
Fix: read the full stack trace in Litium Insights (**Analytics > Dashboard > App Logs**) to find the frame, then set
`CultureInfo.CurrentCulture` and `CurrentUICulture` explicitly at the start of the job, from the channel or website it
works for or a fixed culture, before any culture-dependent lookup, formatting or parsing (`references/code-changes.md`,
section 5). Turn the silent null into an explicit exception so the next difference is diagnosed in one run.
Verify: run the job once in the test environment and check its output in Insights, including the exported or imported
files.

**Integration files are missing after go-live.**
Cause: the code still reads a legacy disk path, and no File storage folder is mounted.
Fix: mount the folder with a `type: storage` configuration and read from `/app_storage/<name>`.
Verify: a file dropped over SFTP is visible to the app.

**`apply` reports *unchanged*, or seems to ignore a manifest.**
Cause: a bare directory was passed to `--file` (a glob is required), or a key was misspelled — unknown keys are
ignored without a warning.
Fix: use `<dir>/*.yaml`, then check the result with `litium-cloud app show`.
Verify: `app show` reflects the change you intended.

**Cached data does not match the database after a restore.**
Cause: Redis still holds keys written under the old prefix.
Fix: last resort only — set `redis_prefix` to a new value in the platform manifest and apply. It clears the
whole cache, including live carts, so never do it during trading hours without telling the customer.
Verify: the stale values are gone and carts behave normally afterwards.

## Apps, search and domains

**A legacy payment or delivery app cannot be uninstalled, or installing the new one fails because the id exists.**
Cause: the restored database still holds the legacy registration; a normal uninstall fails because the app does
not exist in this environment.
Fix: force-delete it — in the back office (apps page under **Settings**, **Uninstall**, then the force-delete
option in the dialog when it fails), or with the `uninstall-app` action on the platform app with
`id=<legacy-app-id>` and `force=true` (needs `apps/litium-platform/litium-management`). Then install the new
app. If the legacy `IdentityServer` folder was copied into the storage artifact, rebuild the storage artifact
without it first — force-deleting with it present can break the apps in the still-live legacy site.
Verify: the back office apps page lists only the new apps.

**Search returns nothing after the restore.**
Cause: the index was not rebuilt after the database came in.
Fix: run the `rebuild-search-indicies` action on the platform app, or rebuild from **Settings** in the back
office.
Verify: a search for a known product returns it, on every channel.

**Payment callbacks for orders placed before go-live return 404.**
Cause: the old webhook URLs were never mapped in the new Fastly service.
Fix: send the old URL list to Litium support before go-live (`references/support-requests.md`) and verify on
T-0 by triggering a capture or refund from the provider's portal.
Verify: the order updates in the new back office from a call to the *old* URL.

**A custom domain shows a certificate error, the wrong site, or nothing.**
Cause: the certificate is not in Fastly; or the domain is in the CDN but not registered in Litium and mapped to
a channel; or it still belongs to the legacy Fastly service — a domain can only exist on one service at a time.
Fix: certificates come from Litium support. Register the domain with the `add-domain` action and map it to a
channel. For the move, confirm the removal and addition order with support.
Verify: the domain opens the new site over HTTPS and the right channel answers.

## Mail and SFTP

**Mail is not sent, or lands in spam.**
Cause: no SMTP relay app; SMTP keys still point at the legacy server; or SPF was not updated for the new sender.
Fix: install the SMTP relay app, point the platform's SMTP configurations at its exposed values, and have the
customer update SPF — for the shared Litium mail server, add `include:mail3.litiumdrift.se` to the sending
domain. That server has no TLS and no DKIM and a shared reputation, so business-critical mail relays through the
customer's own service.
Verify: one real transactional mail arrives from the expected sender.

**The external system cannot connect over SFTP.**
Cause: its IP address is not in the app's `ip` allow list (ten entries per app; a single IP, an `a-b` range or a
CIDR network up to /24 each count as one entry), or the wrong password
was handed over.
Fix: add the address, or install a second `litium-sftp` app. Read the host and user from
`litium-cloud app show --app <sftp-app-id>` under **Exposes**, and the password with
`litium-cloud environment secret value --secret <secret-name>` — never over chat or in `MIGRATION.md`.
Verify: the client connects and lists the mounted folders.

## CLI and pipeline

CLI syntax lives in `litium-cloud-cli` (`references/commands.md`, and the `cicd-service-principal` recipe in
`references/workflows.md`); run `litium-cloud <group> <command> --help` for the exact flags.

**`litium-cloud auth login` fails with "Your connection is not private".**
Cause: an HSTS rule for `localhost` in the browser blocks the sign-in callback — usually left behind by a local
ASP.NET Core site started with HSTS enabled.
Fix: delete the `localhost` entry from the browser's HSTS store, then sign in again.
Verify: the sign-in completes and the CLI prints the signed-in account.

**`dotnet tool update` fails with NU1301 or asks for credentials.**
Cause: the stored credentials for the Litium NuGet feed are missing or stale. They are the *documentation
account* credentials, not the Litium Account.
Fix: remove the Litium NuGet source, add it again with working credentials, then update the tool.
Verify: `litium-cloud --version` prints a version.

**Every command fails with "Could not connect to server."**
Cause: the CLI configuration points at the service principal certificate by a **relative** path, so any command
run from another directory cannot find it — the real error is only visible with `-d`.
Fix: sign in again with an absolute path to the certificate. In Azure DevOps that is `$(<name>.secureFilePath)`.
Verify: the same command works from another working directory.

**Signing in as a service principal fails in the pipeline.**
Cause: the certificate expired (180 days by default, 365 at most), was revoked by a renewal, lost its line
breaks in the secret, or the id does not match the certificate.
Fix: check the certificate dates with `service-principal show`, then `service-principal renew` and replace the
pipeline secret immediately — renewing revokes every other active certificate.
Verify: the pipeline signs in and `artifact create` succeeds.

**A command says access denied, or a resource you know exists is not found.**
Cause: no role on that resource. A resource you may not read is reported as missing, so the two look the same. A
service principal inherits nothing from the person who created it.
Fix: check the grants with `app access-control show` and have an owner grant the missing role; the four minimum
pipeline roles are in the `cicd-service-principal` recipe.
Verify: the command succeeds as that identity.
