# Manifests for a migrated site

**Always start from the downloaded template and merge your values into it. Never author a manifest from
memory.**

```bash
litium-cloud marketplace manifest --app <app-type> -f <file>.yaml     # a new app
litium-cloud app show --app <app-id> -o manifest > <app-id>.yaml      # an app already installed
```

The template is the authority for what an app type accepts: required properties come uncommented, optional ones
commented out with their description. The snippets below are **fragments to merge**, not files to copy whole.
Every key in them appears in the manifest reference or the app's own page
(https://docs.litium.dev/cloud/serverless/reference/manifest.md and the pages under `apps/`). Anything you find
in a template that is not shown here: verify it in the downloaded template, and say so rather than guessing.

Unknown keys are ignored silently, so a typo is never reported — check the result with `litium-cloud app show`
after every apply.

## Where the files live

One file per app under `deploy/serverless/<env>/` in the customer repo, committed. Same file names and same
content in test and production; only artifact ids and the values behind `secretRef` differ. `apply` takes a
glob, not a bare directory: `litium-cloud apply -f 'deploy/serverless/test/*.yaml'`. Files are applied in match
order, so prefix them (`10-`, `20-`) when an app must exist before the app that references it. A single file can
hold several documents separated by a line containing only `---`, applied in order.

## Litium platform

The migration manifest. `sql_backup_file` and `storage_backup_file` are what turn an empty install into the
customer's site.

```yaml litium-platform.yaml
kind: app
resource:
  id: litium                                   # the installed app id, what --app takes
spec:
  type: litium-platform
  version: <version>                           # keep the version from the template
  properties:
    - name: artifact                           # dotnet artifact; no-unset, keep it on every apply
      value: artifacts/<dotnet-artifact-id>
    - name: sql_backup_file                    # create-only: restored at install, then cleared
      value: artifacts/<sqlbackup-artifact-id>
    - name: storage_backup_file                # create-only: restored at install, then cleared
      value: artifacts/<storage-artifact-id>
  configurations:
    - name: ERP__PASSWORD                      # -> Erp:Password in appsettings terms
      valueFrom:
        secretRef:
          name: erp-password                   # environment secret, same id in test and production
    - name: LITIUM__ACCELERATOR__SMTP__HOST     # exposed value of the SMTP relay app
      valueFrom:
        appRef:
          name: smtp
          key: smtp_host
    - name: integration                        # mounted as /app_storage/integration
      type: storage
      subPath: integration-directory           # the folder inside the File storage app
      valueFrom:
        appRef:
          name: file-storage
          key: storage_volume
```

- **Artifact references are strings** of the form `artifacts/<id>`. Property values are always strings;
  booleans are `"true"` / `"false"`.
- **Create-only**: applying a new `sql_backup_file` or `storage_backup_file` to an app that already exists does
  nothing. Restoring a different backup means uninstalling and reinstalling the app (`litium-cloud-cli` recipe
  `restore`).
- **`artifact` is no-unset**: after the first install, new code goes out with `app deploy`, and `apply` keeps
  the deployed artifact. Do not fight it by re-applying an old id.
- **`redis_prefix`** (optional, for example `rev2`) prefixes every Redis key. Changing it clears the cache,
  including live carts, so it is a last resort for a Redis/database mismatch — not something to set routinely
  during a migration.
- The secret must exist **before** you apply: `litium-cloud environment secret create` first,
  `litium-cloud environment secret list` to confirm. An environment secret wins over a subscription secret with
  the same id.

### Configuration naming

`type` is `environment` (default), `file` or `storage`.

| `type` | Where the app finds it |
|---|---|
| `environment` | An environment variable, name uppercased |
| `file` | A file in `/app_secrets/`, name lowercased; a dot in the name is only allowed for this type |
| `storage` | A directory `/app_storage/<name>/` backed by the storage app's `subPath` |

For .NET apps, a double underscore is the section separator: `LITIUM__ACCELERATOR__SMTP__HOST` is read as
`Litium:Accelerator:Smtp:Host`. This is how every value from a legacy `appsettings.<Env>.json` or a config
transform comes back — only `appsettings.json` and `appsettings.production.json` are loaded in the cloud.

## File storage

```yaml file-storage.yaml
kind: app
resource:
  id: file-storage
spec:
  type: file-storage
  version: <version>
  properties:
    - name: backup_file                        # create-only, optional; omit for empty storage
      value: artifacts/<storage-artifact-id>
```

Requires the App Cloud agreement. It exposes `storage_volume`, which is what the platform app and the sFTP apps
reference. Media is **not** stored here — media comes back through the platform app's `storage_backup_file`.

## Litium sFTP

One app per external user: separate credentials, separate folders, at most five IP addresses each.

```yaml sftp-erp.yaml
kind: app
resource:
  id: sftp-erp
spec:
  type: litium-sftp
  version: <version>
  properties:
    - name: ip                                 # comma-separated, or a block with one per line
      value: <ip-address-1>,<ip-address-2>
  configurations:
    - name: integration                        # the folder name inside the sFTP account
      subPath: integration-directory           # the folder inside the File storage app
      type: storage
      valueFrom:
        appRef:
          name: file-storage
          key: storage_volume
```

Exposes `sftp_hostname`, `sftp_username` and `sftp_password` (a secret reference — read the value with
`litium-cloud environment secret value --secret <secret-name>`). Mount the same `subPath` in the platform
manifest so the app reads what the external system drops.

## SMTP relay

```yaml litium-smtp.yaml
kind: app
resource:
  id: smtp
spec:
  type: litium-smtp
  version: <version>
  properties:                                  # omit all three to use the Litium mail server
    - name: smtp_host
      value: <host:port>
    - name: smtp_username
      value: <username>
    - name: smtp_password
      value: <password>                        # reference a secret instead of writing it here
```

Exposes `smtp_host`, `smtp_port`, `smtp_auth_username` and `smtp_auth_password`, and those stay the same
whether it relays through Litium or the customer's own service — so the sending app's manifest does not change
if the customer switches provider later.

## Litium CDN domain

One app per domain. It cannot be updated: to change the domain or its target, delete the app and install it
again.

```yaml domain-www.yaml
kind: app
resource:
  id: domain-www-example-com
spec:
  type: litium-cdn-domain
  version: <version>
  properties:
    - name: cluster_domain                     # the target app's exposed internal_domain_name
      valueFrom:
        appRef:
          name: litium                         # the Litium platform app id, or a storefront app id
          key: internal_domain_name
    - name: public_domain
      value: <www.example.com>                 # must not end in .litium.app
```

This is only the CDN half. The domain must also be registered inside Litium with the `add-domain` action on the
platform app and mapped to a channel in the back office, and a certificate for it must exist in Fastly.

## Payment or delivery app

```yaml <provider>.yaml
kind: app
resource:
  id: <app-id>
spec:
  type: <app-type>                             # from: litium-cloud marketplace list --filter payment
  version: <version>                           # ALWAYS pin it
```

Leaving `version` out installs the default version, and that default changes over time — a migrated site would
then pick up an untested payment app on the next apply. Pin it, and change it deliberately. Installing the app
in the environment only makes it run; it still has to be installed into Litium itself and given the provider's
configuration file (`references/environment-setup.md`, step 6).

## Keeping test and production identical

- Same file names, same `resource.id`, same `type`, same `version`.
- Everything that differs is an environment secret with the **same id** in both environments, referenced with
  `secretRef`. Nothing environment-specific is written as a literal `value`.
- Artifact ids differ; they are the one thing you edit per environment, and at go-live all three of the
  platform's artifact ids change in one commit.
- Export with `app show -o manifest` after any manual change in the Portal, and commit it, or the next
  environment build silently loses it. The export contains no secret values.

## Never

- Never put a secret value, connection string or certificate content in a manifest.
- Never set `ASPNETCORE_ENVIRONMENT` to `Development`; the app does not start.
- Never use `action: delete` on `kind: app` — it is not implemented; use `app delete` and get an explicit yes
  first.
