# `litium-cloud` command reference

Every command below exists in CLI 2.10.1. Verified against `--help`; the installed `--help` is always
the final authority on flags. Docs: `/cloud/serverless/cli/overview` and the per-group pages.

**Convention in this file.** "Starts a job?" means the command returns a job id and exits 0 while the
work continues asynchronously — follow it with `status logs --job <job-id> --follow`.
`--subscription` (`--sub`) and `--environment` (`--env`) are needed only when they cannot be resolved
from a [context](#context); they are omitted from the tables to keep them readable.

## Global options

Accepted by every command:

| Option | Meaning |
|---|---|
| `-o`, `--output <DEFAULT\|JSON\|MANIFEST>` | Output format. `DEFAULT` = human tables, `JSON` = machine-readable, `MANIFEST` = YAML manifest |
| `-d`, `--diagnostics` | Diagnostic output including full exception details |
| `-?`, `-h`, `--help` | Help for the command or group |

Only directly after `litium-cloud`:

| Option | Meaning |
|---|---|
| `--version` | Print the CLI version |
| `--info` | Print version, runtime and platform information (include in a support ticket) |

`-o manifest` works only on the `show` commands that have a manifest equivalent: `app show`,
`artifact show`, `artifact artifact-type show`, `environment show`, `location show`,
`marketplace show`, `subscription show`. Elsewhere it fails.

Use `-o json` for anything a script parses; it never prints progress bars or table borders.

## Directives, environment variables, exit codes, CI

A directive goes in square brackets before the command name:

```bash
litium-cloud [workspace:prod] subscription list
```

`workspace` picks the cloud instance. **Partners use `prod`, which is also the default**, so the
directive is effectively never needed.

| Variable | Meaning |
|---|---|
| `LC_CLI_WORKSPACE` | Workspace when no directive is given. Partners: leave unset (`prod`) |
| `LC_CLI_URL` | Full Cloud API endpoint URL, overrides `LC_CLI_WORKSPACE`. Partners never set it |
| `LC_CLI_VERBOSE` | `true` = verbose output, same as `-d` |
| `LC_CLI_CACHE_DIR` | Folder for the token cache and the global context file. Set it to give a build agent its own isolated cache |

| Exit code | Meaning |
|---|---|
| `0` | Succeeded — for a job-starting command, means *queued*, not *finished* |
| `1` | Failed, with an error message |
| `127` | Command line could not be parsed; parse errors and help are printed |

**CI detection.** The CLI treats the session as non-interactive when any of `CI`, `TF_BUILD`,
`BUILD_BUILDID`, `JENKINS_URL`, `TEAMCITY_VERSION`, `APPVEYOR`, `TRAVIS`, `BUDDY`, `CODEBUILD_CI` or
`BuildRunner` is set. It then never prompts (a command needing confirmation aborts with
*Aborting destructive operation. Use --auto-yes option…*), never opens a browser
(*Unauthenticated. Interactive login not possible.*), and turns progress indication off.

## auth

| Command | Required options | Notable options | Starts a job? | Notes |
|---|---|---|---|---|
| `auth login` | — | `--service-principal`, `--username <sp-id>`, `--certificate <path>` | No | Without options, opens a browser for the Litium Account. The service-principal sign-in is remembered: the id and certificate path are stored and later commands renew the token by themselves |
| `auth logout` | — | — | No | Clears cached accounts and tokens |
| `auth show` | — | — | No | Prints `Current user: <email>` or `Current service-principal: <name>`. Prints `Not logged in.` and exits `1` — a good script guard |
| `auth permission` | — | `--email <email>`, `--filter <text>` | No | Roles of the signed-in account, or of `--email`. Columns: Role id, Role name, Document id |

The certificate must contain public **and** private key, PEM or PFX, no password protection.

## context

Stores a default subscription and environment in `.litium-cloud.config`. Local file (current folder,
searched upwards through parents) beats the global file.

| Command | Required options | Notable options | Starts a job? | Notes |
|---|---|---|---|---|
| `context set` | — | `--subscription`/`--sub`, `--environment`/`--env`, `--global` | No | Accepts an id or a unique partial name. Set the subscription first or in the same command, or it fails |
| `context show` | — | `--global` | No | Unset values print as `(not set)`. `-o json` gives both values |
| `context unset` | — | `--global` | No | Clears both values; the file remains with empty values |

## subscription

Subscriptions are created by Litium; there is no create command.

| Command | Required options | Notable options | Starts a job? | Notes |
|---|---|---|---|---|
| `subscription list` | — | `--filter <text>` | No | Columns: Subscription id, Name, Description (+ State when disabled). Empty list usually means no access granted yet |
| `subscription show` | — | `--subscription` | No | Summary plus Environments and Products tables. Supports `-o manifest` |
| `subscription secret …` | see [secrets](#secret-subcommands) | | | Secrets shared by every environment in the subscription |
| `subscription access-control …` | see [access control](#access-control-subcommands) | | | A grant here applies to every environment, app and secret below, unless inheritance is disabled |

## environment

| Command | Required options | Notable options | Starts a job? | Notes |
|---|---|---|---|---|
| `environment list` | `--subscription` | `--filter <text>` | No | Columns: Id, Production, Environment name, Location |
| `environment show` | — | `--environment` | No | Summary plus an Apps table. Supports `-o manifest` |
| `environment create` | `--name`, `--location`, `--subscription` | `--environment` (explicit id), `--description`, `--production`, `--set-context` | **Yes** | Id is generated from `--name` unless `--environment` is given. Location cannot be changed later |
| `environment update` | `--environment`, `--subscription` | `--name`, `--description`, `--production`, `--non-production` | **Yes** | Restart the installed apps after changing the production flag |
| `environment delete` | `--environment`, `--subscription` | — | **Yes** | **No confirmation prompt.** Deletes every app, database, storage, domain and secret in the environment |
| `environment secret …` | see [secrets](#secret-subcommands) | | | Environment secrets override subscription secrets with the same id |
| `environment access-control …` | see [access control](#access-control-subcommands) | | | The usual scope for giving a developer or a pipeline access |

## app

There is **no `app create` / `app install`** — install with `apply`.

| Command | Required options | Notable options | Starts a job? | Notes |
|---|---|---|---|---|
| `app list` | — | `--filter <text>` | No | Columns: Id, Name, Description (+ State when paused, + Deprecated for a deprecated version) |
| `app show` | `--app` | — | No | State, type, version, plan, plus Configurations, Exposes and Properties tables. The *artifact* property is what is deployed. Supports `-o manifest` |
| `app deploy` | `--app`, `--artifact` | — | **Yes** | Sets the app's artifact-reference property and rolls out. Repeat `--artifact` as `<property-name>=<artifact-id>` for extra references; `--artifact backup=` clears one. Artifact type must match the app |
| `app restart` | `--app` | `--wait` | **Yes** | `--wait` blocks until the job finishes and is unique to this command |
| `app pause` | `--app` | `--auto-yes` | **Yes** | Prompts `Pausing an app will make the app inaccessable. Do you want to continue?` A paused app is unreachable; runtime is not billed, persistent resources still are; deploys are rejected |
| `app resume` | `--app` | — | **Yes** | Starts the app again with the same resources |
| `app plan` | `--app`, and one of `--plan` / `--unset` | — | **Yes** | Plan ids from `marketplace show --app <app-type>`. Changing a plan restarts the app. Public apps answer that plans are not used |
| `app action` | `--app`, `--action` | `--property <name>=<value>` (repeatable) | **Yes** | Prefix a value with `@` to read it from a file: `--property script=@migration.sql`. Booleans are `true`/`false`. Actions and their properties come from `marketplace show --app <app-type> --version <v>` |
| `app job` | `--app` | `--limit <n>` (default `5`, `-1` = all), `--reverse` | No | Columns: Job id, Action, Created by, Created at, Started at (+ Completed/Cancelled at). Oldest first unless `--reverse` |
| `app delete` | `--app` | `--auto-yes` | **Yes** | Prompts `Deleting an app is a destructive operation…`. Deletes the app **and its data**, including databases and storage. Delete dependent apps first |
| `app access-control …` | see [access control](#access-control-subcommands) | | | Access to one app only |

## apply

A single command, not a group. Declarative: for each document it looks up `resource.id` and creates,
updates, deletes or reports *unchanged*.

| Command | Required options | Notable options | Starts a job? | Notes |
|---|---|---|---|---|
| `apply` | `-f`, `--file <file-or-glob>` | `--subscription`, `--environment`, `--app` (for `appAction` manifests) | **Yes**, one per document | A path without a wildcard must be an existing file; a bare directory is not accepted — use `<dir>/*.yaml`. `.git` is always skipped |

| `kind` | Needs on the command line |
|---|---|
| `app` | `--subscription` and `--environment` |
| `appAction` | `--subscription`, `--environment` and `--app` |
| `environment` | `--subscription` |
| `artifact` | Nothing, or `--subscription` to own the artifact. Always creates a new artifact |

Result lines: `… is being created` / `… is being updated` / `… unchanged` / `… is being deleted` /
`… does not exists, skipped`. Kinds are matched case-insensitively; an unsupported kind is skipped
with *Apply is not available for resource of type: `<kind>`*, and a missing `resource` section with
*Can not parse resource type.*

With `-o json`, `apply` prints one `{ "action", "id", "jobId" }` entry per document — `action` is
`create`, `update` or `delete`. This is what a pipeline reads.

`action: delete` works for `kind: environment`. **It is not implemented for `kind: app`** — use
`app delete`. Deleting an artifact with a manifest is not supported — use `artifact delete`.

## artifact

| Command | Required options | Notable options | Starts a job? | Notes |
|---|---|---|---|---|
| `artifact create` | `--artifact-type`, `-f`/`--file`/`--file-path` | `--name`, `--description`, `--no-progress`, `--subscription`, `--no-subscription` | **Yes** | A folder is zipped by the CLI (ignore files apply); a single file is uploaded as it is. Uploads are chunked, so there is no practical size limit. Default name is `<artifact-type> <month-day-time>`. Types built in the cloud also print a `status logs --job <artifact-id>` line |
| `artifact list` | — | `--artifact-type`, `--filter`, `--limit <n>` (default `5`, `-1` = all), `--reverse`, `--no-subscription`, `--all`, `--subscription` | No | Columns: Id, Type, Size, Status, Date, Name. `--limit` counts **per artifact type**. Oldest first unless `--reverse` |
| `artifact show` | `--artifact` | — | No | Name, size, type, status, created/ready dates. Status values: `Initiated`, `Uploading`, `Processing`, `Ready`, `Failed`. Supports `-o manifest` |
| `artifact download` | `--artifact`, `-f`/`--file <path>` | — | No | `<path>` includes the file name |
| `artifact delete` | `--artifact` | — | No | Never delete an artifact an app still references or that you might roll back to |
| `artifact artifact-type list` | — | `--deprecated` | No | **The path is `artifact artifact-type list`**, not `artifact-type list` |
| `artifact artifact-type show` | `--artifact-type` | — | No | Name, description, properties and the manifest the type expects. Supports `-o manifest` |
| `artifact access-control …` | see [access control](#access-control-subcommands) | | | Access to one artifact, for example to let a principal deploy an artifact it did not upload |

For an artifact type built in the cloud, the **artifact id doubles as the id of its build job**:
`litium-cloud status logs --job <artifact-id>`.

Some types read a manifest from the upload: `litiumcloud.manifest.json` or `package.json` (values may
sit under a `litium-cloud` key), from the root of the folder or of the ZIP.

## marketplace

| Command | Required options | Notable options | Starts a job? | Notes |
|---|---|---|---|---|
| `marketplace list` | — | `--filter <text>`, `-d`/`--details`, `--deprecated`, `--prerelease` | No | `--details` prints a section per app with pricing, a Plans table and a Versions table. **This replaces the non-existent `app search`** |
| `marketplace show` | `--app <app-type>` | `--version <version>`, `--deprecated` | No | Without `--version`: the app, its plans and versions. With `--version`: that version's restrictions, **Actions** (properties and permissions per action) and configuration. Supports `-o manifest` |
| `marketplace manifest` | `--app <app-type>` | `--version <version>`, `-f`/`--file <file>` | No | Downloads a commented template: required properties uncommented, optional ones commented out with descriptions. Repeat `--app`, or write `<app-type>@<version>`, to put several manifests in one file separated by `---`; `--version` cannot be combined with several `--app` options. Without `-f` it prints to the terminal |

`marketplace manifest` is the correct starting point for **every** manifest.

## status

| Command | Required options | Notable options | Starts a job? | Notes |
|---|---|---|---|---|
| `status show` | `--job` | — | No | Parent status (`pending` / `in progress` / `completed`), Action, Created by, Insights trace id, then **Details** with one block per child (`pending` / `in progress` / `succeeded` / `failed` / `cancelled`). A parent reads `completed` even when a child failed |
| `status logs` | `--job` | `-f`, `--follow` | No | Prints the job's console output in order, errors in red; `Retry: <n>` marks a retried attempt. `Ctrl+C` stops following. Logs live on the child job that did the work |

## query

Searches across every subscription you can see.

| Command | Required options | Notable options | Starts a job? | Notes |
|---|---|---|---|---|
| `query app` | — | `--app <app-type>[@<version>][#plan=<plan-id>]` | No | Answers "which environments still run the old version?". Prints `No search result` when nothing matches |
| `query environment` | — | `--name`, `--location`, `--production`, `--non-production` | No | Columns: Subscription, Id, Production, Environment name, Location |
| `query jobs` | — | `-co`/`--completed`, `-ca`/`--cancelled`, `-c`/`--child`, `-i`/`--interval <interval>`, `--subscription`, `--environment` | No | Only queued and running jobs by default. `--completed`/`--cancelled` limit to today unless `--interval` is given |

`--interval` takes plain language: `today`, `yesterday`, `this month`, `last 3 days`, `last week`,
`last month`, `last year`, and ranges such as `'7 days to 2 days'`. The headline above the table
repeats the period the CLI actually used.

## role

| Command | Required options | Notable options | Starts a job? | Notes |
|---|---|---|---|---|
| `role list` | — | `--filter <text>` | No | Columns: Id, Role name. The **Id** value is what goes into `--role`. Authoritative for your account |
| `role show` | `--role` | — | No | Name, description, an **Assignable scopes** table and a **Permissions** table. A role can only be granted on a listed scope |

## group

| Command | Required options | Notable options | Starts a job? | Notes |
|---|---|---|---|---|
| `group list` | — | `--filter <text>`, `--for-email <email>` | No | Columns: Id, Group name. `--for-email` lists the groups a user or service principal belongs to |
| `group show` | `--group` | — | No | Name, description and a Members table (Email, Type) |
| `group create` | `--name` | `--description`, `--member <email>` (repeatable) | No | Prints the group id — that id is what you pass as `--email` when granting roles |
| `group update` | `--group` | `--name`, `--description`, `--member` (repeatable) | No | `--member` **replaces** the member list; use the member commands to change one person |
| `group delete` | `--group` | — | No | Removes the access everyone had through the group |
| `group member add` | `--group`, `--email` | — | No | The member gets every role the group has, everywhere |
| `group member remove` | `--group`, `--email` | — | No | |
| `group access-control …` | see [access control](#access-control-subcommands) | | | Controls who may administer the group itself |

Members are users and service principals only — a group cannot contain another group. A member has to
sign out and in again for a new membership to take effect.

## service-principal

| Command | Required options | Notable options | Starts a job? | Notes |
|---|---|---|---|---|
| `service-principal list` | — | `--filter <text>` | No | Columns: Id, Name, Description, Expires at (`N/A` when there is no certificate). Check it before a release |
| `service-principal show` | `--service-principal` | — | No | Name, description and a Certificates table: Id, Not before, Not after, Revoked on |
| `service-principal create` | `--name` | `--expires <days>` (default `180`, max `365`), `-f`/`--file <path>` | No | The private key is shown **once** and never stored by Litium. A `.pfx` name writes PKCS #12; any other name writes PEM. Without `-f` it prints to the terminal |
| `service-principal renew` | `--service-principal` | `--expires <days>` (default `180`, max `365`), `-f`/`--file <path>` | No | **Missing from `service-principal --help` in 2.10.x but it works.** Issues a new certificate and **revokes every other active certificate**. Id and roles are unchanged. `update` is a deprecated alias |
| `service-principal delete` | `--service-principal` | — | No | Every certificate stops working |
| `service-principal access-control …` | see [access control](#access-control-subcommands) | | | Who may administer the principal, for example renew its certificate |

A new service principal has **no roles**. Grant them explicitly with the access-control commands,
passing the principal id as `--email`.

## location

| Command | Required options | Notable options | Starts a job? | Notes |
|---|---|---|---|---|
| `location list` | — | `--filter <text>` | No | Columns: Id, Name, Description, Type. The Id is `--location` for `environment create` |
| `location show` | `--location` | — | No | Name, description, type. Supports `-o manifest` |

An environment stays in the location it was created in. Use the same location for test and production.

## Secret subcommands

Identical on `subscription secret` and `environment secret`; the only difference is the level and
that `environment secret list` also shows inherited subscription secrets.

| Command | Required options | Notable options | Starts a job? | Notes |
|---|---|---|---|---|
| `… secret list` | — | `--filter <text>` | No | Columns: Id, Description (+ *Value source* on an environment: *Environment*, *Subscription*, *Environment (override subscription)*). Values are never printed |
| `… secret create` | `--secret`, and one of `-v`/`--value`, `--text-value <path>`, `--binary-value <path>` | `--description` | No | Max 25 KB. Without a value option it stops with `Missing secret value.` Prefer the file options — `-v` lands in shell history and build logs |
| `… secret update` | `--secret`, and one of the three value options | `--description` | No | Restart the apps that use the secret afterwards |
| `… secret value` | `--secret` | `-f`/`--file <path>` | No | Prints the value **in clear text**, no trailing newline. A binary secret must go to a file. `-o json` is not supported |
| `… secret delete` | `--secret` | — | No | An app referencing a deleted secret fails the next time it starts |
| `… secret access-control …` | `--secret` plus the access-control options | | | Per-secret access, for example one team reading one secret |

Examples:

```bash
litium-cloud subscription secret create --secret license-key --text-value ./license.txt --description "Shared license key"
litium-cloud environment secret create  --secret erp-password --text-value ./erp.txt
litium-cloud environment secret list
litium-cloud environment secret value  --secret erp-password -f ./erp.txt
```

## Access-control subcommands

The same five verbs exist on `subscription`, `subscription secret`, `environment`,
`environment secret`, `app`, `artifact`, `group` and `service-principal`. Only the option naming the
resource differs (`--secret`, `--app`, `--artifact`, `--group`, `--service-principal`; subscriptions
and environments use `--subscription` / `--environment`).

| Command | Required options | Notable options | Starts a job? | Notes |
|---|---|---|---|---|
| `… access-control add` | `--email`, `--role` | | No | `--email` is a user email, a group id or a service principal id. `--role` is a role id from `role list`. Prints `Access control is added.` |
| `… access-control remove` | `--email`, `--role` | | No | Only removes an entry granted **on this resource**. Remove an inherited entry where it was granted |
| `… access-control show` | — | `-d`/`--details`, `--filter <text>` | No | Prints `Inheritance: Yes\|No` and then Role id, Role name, Member Type, Member, Member name, Inherited. `--details` expands group members |
| `… access-control enable-inheritance` | — | | No | Inherit from the parent again; explicit entries stay |
| `… access-control disable-inheritance` | — | `--convert <true\|false>` (default `true`) | No | `true` copies the inherited entries as explicit ones first. **`--convert false` drops them all** and can lock you out — grant yourself an explicit role first and check with `show` |

Example, the four grants a deployment pipeline needs:

```bash
litium-cloud subscription access-control add --subscription <subscription-id> --email <service-principal-id> --role subscription/reader
litium-cloud subscription access-control add --subscription <subscription-id> --email <service-principal-id> --role artifact/creator
litium-cloud environment  access-control add --subscription <subscription-id> --environment <environment-id> --email <service-principal-id> --role environment/reader
litium-cloud app          access-control add --subscription <subscription-id> --environment <environment-id> --app <app-id> --email <service-principal-id> --role appresource/writer
```
