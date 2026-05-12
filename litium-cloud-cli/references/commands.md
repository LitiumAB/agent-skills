# Litium Cloud CLI — Command Reference

All commands follow the pattern: `litium-cloud <command> <subcommand> [options]`

Run `litium-cloud <command> --help` for the latest flags. This file serves as a baseline; the CLI itself is the authoritative source.

## Global options (all commands)

| Option | Description |
|---|---|
| `-o, --output` | Output format: `Default` (pretty) or `Json` |
| `-d, --diagnostics` | Enable verbose diagnostic logging |
| `--subscription, --sub` | Target subscription ID (can be set via context) |
| `--environment, --env` | Target environment ID (can be set via context) |
| `--no-progress` | Disable progress spinners |
| `--wait` | Block until async job finishes (where supported) |
| `--auto-yes` | Skip confirmation prompts |
| `--version` | Show CLI version |

---

## apply

Apply a YAML manifest to install or update apps in an environment.

```bash
litium-cloud apply -f <manifest.yaml> [--subscription <id>] [--environment <id>]
```

| Option | Description |
|---|---|
| `-f, --file` | **(required)** Path to the YAML manifest file |
| `--subscription` | Target subscription |
| `--environment` | Target environment |
| `--app` | Scope to a specific app ID |

**Example:**
```bash
litium-cloud apply -f litium-platform.yaml --sub my-sub --env my-env
```

---

## app

Manage applications installed in an environment.

### app list
```bash
litium-cloud app list [--filter <text>] [--subscription <id>] [--environment <id>]
```

### app show
```bash
litium-cloud app show --app <app-id> [--subscription <id>] [--environment <id>]
litium-cloud app show --app <app-id> -o manifest > myapp.yaml   # export as YAML
```

### app deploy
Deploy an artifact to an installed app.
```bash
litium-cloud app deploy --app <app-id> --artifact <artifact-id> [--subscription <id>] [--environment <id>] [--wait]
```

### app action
Run a named action against an app (e.g. create backup, rebuild search index).
```bash
litium-cloud app action --action <action-name> --app <app-id> [--property <key=value>] [--subscription <id>] [--environment <id>]
```

Common actions for Litium Platform app:
| Action | Description |
|---|---|
| `backup-database` | Create a database backup artifact |
| `backup-storage` | Create a storage backup artifact |
| `rebuild-search-indicies` | Rebuild Elasticsearch indices |
| `install-app` | Install a backoffice app (payment/delivery) |
| `uninstall-app` | Uninstall a backoffice app |
| `configure-app` | Configure a backoffice app |
| `add-domain` | Add a domain to the platform |
| `remove-domain` | Remove a domain from the platform |
| `execute-database-script` | Run a SQL script |

### app pause / resume
```bash
litium-cloud app pause  --app <app-id>
litium-cloud app resume --app <app-id>
```

### app restart
```bash
litium-cloud app restart --app <app-id>
```

### app delete
```bash
litium-cloud app delete --app <app-id>
```

### app plan
Change the resource plan (CPU/memory allocation) for an app.
```bash
litium-cloud app plan --app <app-id> --plan <plan-id>
```

### app access-control
Manage who can access an app.
```bash
litium-cloud app access-control add    --email <email> --role <role> --app <id>
litium-cloud app access-control remove --email <email> --role <role> --app <id>
litium-cloud app access-control show   --app <id>
```

---

## artifact

Manage deployment artifacts.

### artifact create
```bash
litium-cloud artifact create \
  --file-path <path-to-folder-or-zip> \
  --artifact-type <type> \
  [--name "<name>"] \
  [--description "<desc>"] \
  [--subscription <id>]
```

Artifact types: `dotnet`, `nextjs`, `nodejs`, `nuxtjs`

For backup artifacts, use `app action --action backup-database` or `backup-storage` instead.

**Example:**
```bash
litium-cloud artifact create --name "v1.2.0" --file-path ./publish/ --artifact-type dotnet --sub my-sub
```

### artifact list
```bash
litium-cloud artifact list [--filter <text>] [--subscription <id>] [--limit <n>]
```

### artifact show
```bash
litium-cloud artifact show --artifact <artifact-id>
```

### artifact download
```bash
litium-cloud artifact download --artifact <artifact-id> --file <output-path>
```

### artifact delete
```bash
litium-cloud artifact delete --artifact <artifact-id>
```

---

## auth

Manage authentication.

### auth login
```bash
# Interactive (browser)
litium-cloud auth login

# Service principal (CI/CD)
litium-cloud auth login --service-principal --username <sp-email> --certificate <cert.pem>
```

### auth logout
```bash
litium-cloud auth logout
```

### auth show
```bash
litium-cloud auth show
```

---

## context

Manage default subscription/environment to avoid repeating flags.

### context set
```bash
litium-cloud context set --subscription <id> --environment <id>   # current directory
litium-cloud context set --subscription <id> --global             # global (all directories)
```

### context show
```bash
litium-cloud context show
```

### context unset
```bash
litium-cloud context unset
```

---

## environment

Manage cloud environments.

### environment create
```bash
litium-cloud environment create \
  --name <name> \
  --location <location-id> \
  [--subscription <id>] \
  [--production] \
  [--description "<text>"] \
  [--set-context]
```

The `--production` flag marks the environment as production and applies production-grade resources. **All non-production environments get `noindex`/`nofollow` robots directives automatically.**

### environment list
```bash
litium-cloud environment list [--subscription <id>]
```

### environment show
```bash
litium-cloud environment show [--subscription <id>] [--environment <id>]
```

### environment update
```bash
litium-cloud environment update --name <new-name> [--production] [--non-production]
```

Use `--non-production` to demote a production environment.

### environment delete
```bash
litium-cloud environment delete [--subscription <id>] [--environment <id>]
```

### environment access-control
```bash
litium-cloud environment access-control add    --email <email> --role <role>
litium-cloud environment access-control remove --email <email> --role <role>
litium-cloud environment access-control show
```

---

## group

Manage user groups for access control.

### group create
```bash
litium-cloud group create --name <name> [--description "<text>"] [--member <email>]
```

### group list
```bash
litium-cloud group list [--filter <text>]
```

### group member add / remove
```bash
litium-cloud group member add    --group <id> --email <email>
litium-cloud group member remove --group <id> --email <email>
```

### group access-control
```bash
litium-cloud group access-control add --email <email> --role <role> --group <id>
```

---

## location

List available deployment regions.

```bash
litium-cloud location list
litium-cloud location show --location <id>
```

Use the location ID from `location list` when creating an environment.

---

## marketplace

Browse and download app definitions.

### marketplace list
```bash
litium-cloud marketplace list [--filter <text>] [--prerelease] [--limit <n>]
```

### marketplace show
```bash
litium-cloud marketplace show --app <app-id> [--version <ver>]
```

### marketplace manifest
Download the YAML manifest template for an app.
```bash
litium-cloud marketplace manifest --app <app-id> -f <output.yaml>
```

Common app IDs:
| App ID | Description |
|---|---|
| `litium-cdn` | Litium CDN (Fastly) — install first |
| `litium-platform` | Litium Platform (.NET backend) |
| `litium-insights` | Litium Insights (logs and metrics) |
| `litium-storefront` | Litium Storefront (React headless frontend) |
| `litium-storefront-proxy` | Storefront Proxy (traffic routing) |
| `litium-sftp` | sFTP access to file storage |
| `smtp-relay` | SMTP relay for email |

---

## role

View available roles.

```bash
litium-cloud role list
litium-cloud role show --role <id>
```

---

## service-principal

Manage service accounts for CI/CD automation.

### service-principal create
```bash
litium-cloud service-principal create \
  --name <name> \
  --file <output-cert.pem> \
  [--expires <days>]   # default 180, max 365
```

After creation, the service principal is assigned an email address (shown in `service-principal list`).

### service-principal list
```bash
litium-cloud service-principal list [--filter <text>]
```

### service-principal renew
```bash
litium-cloud service-principal renew --service-principal <id> --file <new-cert.pem> [--expires <days>]
```

### service-principal delete
```bash
litium-cloud service-principal delete --service-principal <id>
```

---

## status

View job status and logs.

### status show
```bash
litium-cloud status show --job <job-id>
```

### status log
```bash
litium-cloud status log --job <job-id>
```

---

## subscription

View subscriptions.

```bash
litium-cloud subscription list [--filter <text>]
litium-cloud subscription show [--subscription <id>]
```

### subscription access-control
```bash
litium-cloud subscription access-control add    --email <email> --role <role> --sub <id>
litium-cloud subscription access-control remove --email <email> --role <role> --sub <id>
litium-cloud subscription access-control show   --sub <id>
```
