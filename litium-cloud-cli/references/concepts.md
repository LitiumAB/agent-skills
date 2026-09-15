# Litium Serverless Cloud — concepts

Condensed from the public documentation. Docs: `/cloud/serverless/concepts`,
`/cloud/serverless/reference/manifest`, `/cloud/serverless/reference/roles-and-permissions`,
`/cloud/serverless/reference/app-actions`.

## Hierarchy

```
subscription                     created by Litium; owns artifacts and subscription secrets
└── environment                  e.g. test, qa, prod; has a location and a production flag
    └── app                      everything that runs: platform, CDN, Insights, storefront, DB, your code
```

- **Subscription** — the customer's agreement with Litium. Litium creates it and grants the first
  users access; after that the partner manages access. It carries the *products* that decide what may
  be installed (for example whether private apps are allowed). There is no command to create one.
- **Environment** — an isolated space inside a subscription. Id is lowercase letters, digits and
  hyphens, and becomes part of every app's system domain, so keep it short. It has a **location**
  (region, immutable after creation) and a **production** flag.
- The **production flag** switches on production behaviour: production resources for apps that support
  it, and search-engine indexing allowed on custom domains. It is also what lets Litium activate a
  dedicated worker node for the platform app's background jobs when they need one (Litium 8.16+, at no
  extra cost, on request — never partner-configurable). Non-production environments, and the `litium.app` system domain in every
  environment, always answer `noindex, nofollow`. Changing the flag requires restarting installed apps.

## Apps

**Public apps** are provided by Litium and included in the subscription (some need a specific
agreement): Litium platform, Litium CDN, Litium CDN domain, Litium Insights, Litium Storefront,
Litium SFTP, the SMTP relay app, payment and delivery apps.
**Private apps** host your own workloads and require the App Cloud agreement: .NET web, Next.js web,
Node.js web, Nuxt.js web, MS SQL, Redis Cache, File storage, Headless Chrome.

Two ids are easy to confuse:

| Term | Example | Where it is used |
|---|---|---|
| **App type** | `litium-platform`, `dotnet-web`, `litium-nextjs-web` | `marketplace list`, `spec.type` in a manifest |
| **Installed app id** | `litium`, `storefront`, `integration-api` | `--app`, `resource.id` in a manifest |

The installed app id must be unique within the environment. Each app type has one or more **versions**;
the default version is used when `spec.version` is omitted, and defaults change over time — pin a
version for anything you care about. Downgrading is not supported: uninstall and install again.

Some types allow only one instance per environment (Litium platform, Litium CDN, Litium Insights,
Litium Storefront).

**App plans** set CPU, memory and replica count. Private apps that run code (.NET web, Next.js web,
Node.js web, Nuxt.js web, Redis Cache, Headless Chrome) require a plan, taken from
`marketplace show --app <app-type>` and set as `spec.plan` or with `app plan`. Plan ids ending in *a*
autoscale on CPU. Litium platform and the Litium storefront apps get a plan assigned automatically.
Storage-based private apps (MS SQL, File storage) have no plan and are billed on disk usage.

Apps are **never** installed with an `app install` command — always `litium-cloud apply -f <manifest>`.

## Artifacts

An artifact is an immutable file uploaded to a **subscription**, so the same artifact can be deployed
to test and later to production. Uploading is the one task with no Portal equivalent.

Live `artifact artifact-type list` output — the partner-facing types are:

| Type | Contains | Downloads as |
|---|---|---|
| `dotnet` | A published .NET application (built locally, then uploaded) | `.zip` |
| `nextjs` | Next.js source, built in the cloud | `.zip` |
| `nodejs` | Node.js source, built in the cloud | `.zip` |
| `nuxtjs` | Nuxt.js v3 source, built in the cloud | `.zip` |
| `sqlbackup` | An MS SQL backup file | `.bak` |
| `storage` | An app's file storage, for example the media folder | `.zip` |

`script-result` (output of `execute-database-script --property file_output=true`) and `db-migration`
(an EF migration bundle for the MS SQL app) also appear. `litium-db-tool` and `redisbackup` are
internal — ignore them.

**Code artifacts** (`dotnet`, `nextjs`, `nodejs`, `nuxtjs`) go into the app's `artifact` property and
can be changed at any time with `app deploy`. **Backup artifacts** (`sqlbackup`, `storage`) go into
`sql_backup_file` / `storage_backup_file`, which are **create-only**: they are applied only when the
app is installed, so restoring means uninstall + reinstall.

**Retention.** An artifact that was never used by an app is deleted **14 days** after it was created.
An artifact that has been used is deleted **7 days** after the last app stopped using it. An artifact
an installed app currently references is never deleted automatically. Do not treat the cloud as a
release archive — rebuild from source.

**Ignore files.** When `-f`/`--file-path` points at a folder, the CLI zips it and applies rules in
order: everything under the folder, then `.git/**` always excluded, then the lines of `.gitignore`,
`.npmignore` and `.litiumcloudignore` read **from the root of that folder only**. The last matching
rule wins, and a line starting with `!` re-includes. A `.gitignore` in a subfolder has no effect.
When `-f` points at a single file (for example a `.bak` or a `.zip`), it is uploaded as it is.

## Manifests

A manifest is YAML applied with `litium-cloud apply -f <file-or-glob>`. Keys are camelCase; unknown
keys are silently ignored, so verify with `app show` after applying. There is **no `apiVersion` key**.

Kinds: `app`, `environment`, `appAction`, `artifact`. `artifact` only registers a record — use
`artifact create` to upload content, and note that an artifact manifest always creates a new artifact.

A correct, annotated app manifest:

```yaml litium-platform.yaml
kind: app                              # app | environment | appAction | artifact
resource:
  id: litium                           # the installed app id, what you pass to --app
metadata:                              # optional string labels, shown by `app show`
  team: commerce
spec:
  type: litium-platform                # app type from `marketplace list`; required on create
  version: <version>                   # omit on create for the default; omit on update to keep current
                                       # no `plan:` here — Litium platform assigns its own
  description: Customer web            # optional free text
  properties:                          # type-specific settings, values are always strings
    - name: artifact                   # no-unset: omitting it on update keeps the deployed artifact
      value: artifacts/<artifact-id>
    - name: sql_backup_file            # create-only: applied only when the app is installed
      value: artifacts/<artifact-id>
    - name: redis_prefix               # changing it clears the whole Redis cache, including carts
      value: rev1
  configurations:                      # runtime values delivered into the container
    - name: ERP__HOST                  # environment variable (default type), uppercased
      value: https://erp.example.com
    - name: ERP__PASSWORD
      valueFrom:
        secretRef:                     # value of the secret with this id, environment before subscription
          name: erp-password
    - name: ConnectionStrings__DefaultConnection
      valueFrom:
        appRef:                        # a value another installed app exposes, same environment
          name: privatedb              # the installed app id
          key: connection_string       # the exposed key, see `app show` → Exposes
    - name: erp_certificate.pem        # written to /app_secrets/erp_certificate.pem
      type: file
      valueFrom:
        secretRef:
          name: erp-certificate
    - name: exports                    # mounted as /app_storage/exports
      type: storage
      subPath: exports
      valueFrom:
        appRef:
          name: file-storage
          key: storage_volume
actions:                               # optional access-control changes applied after create/update
  - name: acl-add
    value:
      email: <service-principal-id>
      role: appresource/writer
```

Property restrictions the app type declares: **required** (must be set), **create-only** (ignored on
update — reinstall to change), **no-unset** (omitting on update keeps the current value).

`actions` entries: `acl-add`, `acl-remove`, `acl-enable-inheritance`, `acl-disable-inheritance`
(the last takes `convert`, default `true`).

`action: delete` at the top level removes the resource — **works for `kind: environment`, not for
`kind: app`** (use `app delete`). Deleting an artifact with a manifest is not supported either.

Multi-document files are separated by a line containing only `---`, applied in order. Files matched
by a glob are applied in the order the pattern returns them, so prefix them `10-`, `20-` when order
matters. `-f` needs a file or a glob (`manifests/*.yaml`) — a bare directory is not accepted.

## Configurations in the container

| `type` | Where the app finds it |
|---|---|
| `environment` (default) | An environment variable, name uppercased |
| `file` | A file in `/app_secrets/`, name lowercased |
| `storage` | A directory `/app_storage/<name>/`, backed by the storage app's `subPath` |

Names start with a letter, continue with letters, digits and underscores, end with a letter or digit,
max 255 characters. A dot is allowed only when `type: file`.

For .NET apps, a **double underscore maps to a configuration section separator**:
`LITIUM__ACCELERATOR__SMTP__HOST` is read as `Litium:Accelerator:Smtp:Host`, identical to the nested
`appsettings.json` structure. A Litium platform app loads only `appsettings.json` and
`appsettings.production.json`; never set `ASPNETCORE_ENVIRONMENT` to `Development`.

## Secrets

Secrets are stored by the platform, never in a manifest. Max 25 KB per value.

- **Subscription secrets** are visible in every environment of the subscription.
- **Environment secrets** belong to one environment.
- **Precedence: the environment secret wins** when both levels hold the same id.
  `environment secret list` shows the *Value source* column as *Environment*, *Subscription* or
  *Environment (override subscription)*.

Apps read secrets at startup — restart the app after changing one. `secret value` prints in clear
text; never run it in a pipeline whose log is published.

## Context file

`litium-cloud context set --subscription <id> --environment <id>` writes `.litium-cloud.config`.
Without `--global` it lands in the current folder and the CLI searches upwards through parent folders,
so each project folder deploys to its own environment. With `--global` it goes to the CLI application
data folder. A local file wins over the global one. Set the
subscription first or in the same command. The file holds ids, not secrets, but gitignore it anyway.

## Authentication

- **Interactive**: `litium-cloud auth login` opens a browser for a **Litium Account** — the same
  account used for the Portal and Litium Insights, separate from the docs account, requires 2FA.
  Tokens are cached in the Windows credential store, the macOS keychain (service *Litium.CloudAPI*)
  or the Linux keyring.
- **Service principal**: a non-interactive identity with an id in email form, `service.<name>@cloud`.
  It signs in with a certificate: `auth login --service-principal --username <id> --certificate <path>`.
  The CLI generates the key pair locally and Litium never stores the private key.
  Certificates default to **180 days** and are capped at **365 days**.
  **`service-principal renew` revokes every other active certificate on the principal**, so update the
  pipeline secret in the same maintenance window. The id and the roles are unchanged.
  A service principal inherits **nothing** from the person who created it.
  Use an **absolute** certificate path — a relative path stored in the config surfaces later as a
  misleading `Could not connect to server.` from another directory.

## Access control

A **role** is granted to a **principal** (user, group or service principal) on a **scope**
(subscription, environment, app, artifact, secret, group, service principal).

**System roles** — grantable on any resource, covering everything inside it:

| Role id | Portal name |
|---|---|
| `system/owner` | Owner — full access, including granting roles |
| `system/contributor` | Contributor — full access, cannot grant roles |
| `system/writer` | Writer — view and change, not delete or create |
| `system/reader` | Reader — view only |
| `system/acl-manager` | User access manager — manage access without access to the resource |

**Resource roles** follow the pattern `creator` / `reader` / `writer` / `contributor` on one resource
type: `subscription/*` (no creator), `environment/*`, `appresource/*`, `artifact/*`, `secret/*`,
`group/*`, `serviceprincipal/*`. Use them for narrow pipeline access. Read access on a resource does
not imply read access on its parent: an app-scoped principal also needs `environment/reader` and
`subscription/reader`.

**App-specific roles** for actions and Insights:
`apps/litium-platform/backup-operator`, `apps/litium-platform/database-script-execution`,
`apps/litium-platform/litium-management`, `apps/mssql-db/backup-operator`, `apps/mssql-db/script-execution`,
`apps/mssql-db/migration-execution`, `apps/file-storage/backup-operator`,
`apps/litium-insights/insights-logs`, `apps/litium-insights/insights-bi`.

Owner and Contributor cover backups, database migrations and CDN domain changes, but **not** SQL
script execution and **not** the Litium platform management actions (`install-app`, `uninstall-app`,
`configure-app`, `add-domain`, `remove-domain`, `replace-domain`, `rebuild-search-indicies`) — those
always need the app-specific role.

To change access on a resource you need **Owner** or **User access manager** on it; Contributor can
manage the resource but not assign roles.

**Inheritance** passes grants downwards and is on by default. `access-control disable-inheritance`
takes `--convert` (default `true`), which copies the inherited grants as explicit ones first;
`--convert false` drops them and can lock you out. `enable-inheritance` puts inheritance back and
keeps the explicit entries. Some grants are *protected* by Litium and survive inheritance changes.
`role list` is the authoritative list for your account; `role show --role <id>` lists the assignable
scopes and permissions. New roles take effect after the user signs out and in again.

## Jobs

Every change runs as a job. The command prints a 32-hex-character job id and exits 0 as soon as the
job is *queued*. Jobs have child jobs — installing Litium platform has children for the database, the
storage and the app.

The parent line reads `pending`, `in progress` or `completed`. Each entry under **Details** reads
`pending`, `in progress`, `succeeded`, `failed` or `cancelled`. **A parent says `completed` even when
a child failed**, so always read Details. Logs live on the child job that did the work, so run
`status logs` on the child job id when the parent log is empty.

`status show` also prints an **Insights trace id** to search for in Litium Insights, and
`Parent job id` / `Waiting on job` when they apply. For an artifact build job, the artifact id doubles
as the job id.

## Portal vs CLI

The Portal (https://portal.litium.cloud) and the CLI are two front ends to the same platform, signed
in with the same Litium Account.

| Task | Portal | CLI |
|---|---|---|
| Create, duplicate, delete environments | Yes | Yes |
| Install, configure, pause, restart, uninstall apps | Yes | Yes, with manifests and `apply` |
| Run app actions (backups, SQL scripts, domains) | Yes, the app's **Actions** page | Yes, `app action` |
| Secrets, access, groups, service principals | Yes | Yes |
| Follow jobs and read logs | Yes, with live streaming | Yes, `status show` / `status logs` |
| **Upload artifacts** | **No** | **Yes, `artifact create`** |
| Automate from a pipeline | No | Yes, with a service principal |

Litium Insights (application and request logs, metrics) is at https://insights.litium.cloud.
Job logs — installs, deploys, actions — are read with `status logs`, not in Insights.
