# Pipeline migration

Phase 5. Replaces the legacy Azure DevOps build + Web Deploy release with a pipeline that publishes for Linux,
uploads a `dotnet` artifact and deploys it with the CLI. Sources of truth:
https://docs.litium.dev/cloud/serverless/guides/automated-deployments/overview.md, `azure-devops.md`,
`github-actions.md` and https://docs.litium.dev/cloud/serverless/guides/access/service-principals.md.
Starting points: `assets/azure-pipelines.yml` and `assets/github-actions.yml` — byte-identical to the pipelines on
those docs pages. Adapt and test them in the customer's own repository before pointing anything at production.

The pipeline is **rewritten, not patched**. The build stage survives almost unchanged; the release stage is
deleted and replaced by separate CLI steps.

## Step structure

Both assets are the same seven steps, one bash step per task, each with `set -euo pipefail`, inputs from
environment variables (secrets are mapped in `env:`, never spliced into the script body) and a fail-fast exit:

| # | Step | Command | Reads | Passes on |
|---|---|---|---|---|
| 1 | Build and publish | `dotnet publish <web>.csproj --configuration Release --framework <tfm> --os linux --arch x64 --output <publish>` | — | Fails if `litiumcloud.manifest.json` is missing from the publish folder (`Litium.Cloud.Tools.Targets` not referenced) |
| 2 | Install the CLI | `dotnet tool update --global litium.cloud.cli --no-cache`, then put `~/.dotnet/tools` on `PATH` | — | — |
| 3 | Sign in | `litium-cloud auth login --service-principal --username <id> --certificate <absolute path>` | secure file / secret | — |
| 4 | Create the artifact | `litium-cloud artifact create --subscription <id> --artifact-type dotnet --file-path <publish> --name <build number> --no-progress -o json` | — | `data.artifactId` |
| 5 | Wait for the artifact | poll `litium-cloud artifact show --artifact <id> -o json` | `status` | stops on `Ready`; on `Failed` prints `status logs --job <failedJobId>` and exits 1; times out |
| 6 | Deploy | `litium-cloud app deploy --subscription <id> --environment <id> --app <id> --artifact <id> -o json` | — | `jobId` |
| 7 | Wait for the job | poll `litium-cloud status show --job <id> -o json` | `items[].failed` at every depth, then `completedAt` | prints failed items and their `status logs`; exits 1 on failure; times out |
| 8 | Optional production stage | steps 6 and 7 in a template (Azure) or reusable workflow (GitHub), behind an environment approval | the artifact id from step 4 | — |

The docs pages explain each step under "How the pipeline works" and show the production stage in full.

## What the legacy setup does

A **build pipeline** restores from the Litium NuGet feed, runs the `yarn` client builds, builds and publishes
the solution and publishes the output as an Azure DevOps build artifact called `LitiumBuildArtifact`. A
**release pipeline** then extracts that artifact, copies a `license.json` secure file into it, rewrites
`appsettings.json` with a Magic Chunks transform (typically `Litium/Data/ConnectionString`), and runs
`msdeploy.exe` three times against `https://<server>:8172/msdeploy.axd`: stop the app pool, sync the files,
start the app pool.

## Side-by-side mapping

| Legacy step | Serverless Cloud equivalent |
|---|---|
| Restore with the Litium NuGet service connection | Unchanged. The feed is still needed, now also to install the CLI as a .NET tool |
| `yarn install` / `yarn run prod` per client project | Unchanged; keep them before the publish |
| `dotnet publish '*.sln' -c Release -o builds/publish` (solution-wide, Windows) | `dotnet publish <web>.csproj --configuration Release --framework net8.0 --os linux --arch x64 --output publish` — **one project**, the startup web project, targeted at Linux x64. Match `--framework` to the solution's target framework |
| `Litium.Cloud.Tools.Targets` not present | Added to the web project; it writes `litiumcloud.manifest.json` (entry point, .NET version, probe paths) at publish. Step 1 checks that the file exists and fails otherwise. See `references/code-changes.md` |
| `PublishBuildArtifacts@1` → `LitiumBuildArtifact` | Dropped. The publish folder is uploaded straight to Serverless Cloud as a `dotnet` artifact |
| Release stage: **Extract files** | Dropped; nothing is zipped |
| Release stage: **Download secure file** `license.json` + **Copy files** | `license.json` belongs in the artifact. Keep it in the repo, or keep the secure-file download and copy it into the publish folder before the artifact is created |
| Release stage: **Magic Chunks** transform of `appsettings.json` | Dropped. SQL, search and Redis connection settings are injected by the platform; everything else becomes manifest `configurations` and `secretRef` secrets (`references/manifests.md`) |
| — | New (step 2): `dotnet tool update --global litium.cloud.cli --no-cache` |
| — | New (step 3): download the service principal certificate (secure file / repository secret) and `litium-cloud auth login --service-principal --username <service-principal-id> --certificate <absolute path>` |
| — | New (step 4): `litium-cloud artifact create --subscription <id> --artifact-type dotnet --file-path <publish-folder> --name <build number> --no-progress -o json`, read `data.artifactId` |
| — | New (step 5): poll `litium-cloud artifact show --artifact <id> -o json` until `status` is `Ready`, with a timeout |
| PowerShell: msdeploy `recycleApp` StopAppPool | Dropped. Production deployments are rolling; no downtime step exists |
| PowerShell: msdeploy `sync` contentPath with `AppOffline` | Step 6: `litium-cloud app deploy --subscription <id> --environment <id> --app <id> --artifact <artifact-id> -o json`, read `jobId` |
| PowerShell: msdeploy `recycleApp` StartAppPool | Dropped. Step 7 polls `litium-cloud status show --job <job-id> -o json` instead |
| Deployment "succeeded" when msdeploy exited 0 | The job is finished when `completedAt` is set, and succeeded when no entry in `items`, at any depth, has `failed` set to `true`. On failure print `litium-cloud status logs --job <job-id>` for each failed item and fail the pipeline |

**Caveat worth writing into the pipeline:** a parent job reports *completed* as soon as it finishes, whether or
not the work succeeded, and a failed item can sit underneath it — one or more levels down, because items nest
(`items[].items[]`). Checking `completedAt` alone marks a broken deploy as green, and checking only the top level
of `items` misses a failed grandchild. Both assets search every level with
`jq '[.. | objects | select(.failed == true) | .jobId] | unique[]'`; `failed` is only present in the JSON when it
is `true`.

Artifact statuses in the JSON are `Initiated`, `Uploading`, `Processing`, `Ready` and `Failed`, exactly as
written. `nextjs`, `nodejs` and `nuxtjs` artifacts are built after upload, so several minutes in `Processing` is
normal; raise the timeout (`artifactTimeoutMinutes` / `ARTIFACT_TIMEOUT_MINUTES`) rather than the poll interval.
On `Failed`, `artifact show` returns a `failedJobId`; `litium-cloud status logs --job <failedJobId>` prints the
build log. When the field is missing the artifact id doubles as the id of the job that processed it.

## Variable mapping

| Legacy variable | Becomes | Secret |
|---|---|---|
| `PublishName` (msdeploy site) | `app` / `LITIUM_APP` — the installed app id, for example `litium` | No |
| `PublishServer` (`https://<server>:8172/msdeploy.axd`) | `subscription` + `environment` ids | No |
| `PublishUsername` | The service principal id (`service.<name>@cloud`): `LitiumServicePrincipal` in the Azure variable group, `LITIUM_SP_ID` in GitHub | Yes |
| `PublishPassword` | The service principal **certificate**: the Azure secure file `litium-cloud.pem`, or the GitHub secret `LITIUM_SP_CERTIFICATE` | Yes |
| `Source`, `ExtractedSource` | `publishPath` / `PUBLISH_PATH` — the folder `dotnet publish` wrote | No |
| Magic Chunks transformation text (connection string) | Deleted; the platform injects it | — |
| `license.json` secure file | Kept only if the license is not in the repo; it must end up inside the publish folder | Yes |
| Litium NuGet user/password | Kept, still needed for restore and for installing the CLI (`LitiumNuGetUser` / `LitiumNuGetPassword`, or `LITIUM_NUGET_USER` / `LITIUM_NUGET_PASSWORD`) | Yes |
| — | New: `artifactType` / `ARTIFACT_TYPE` (`dotnet`, or `nextjs`, `nodejs`, `nuxtjs` for a storefront) | No |
| — | New: `project`, `targetFramework`, `pollSeconds`, `artifactTimeoutMinutes`, `deployTimeoutMinutes` (upper-case equivalents in GitHub) | No |

Nothing else from the release stage survives. Delete the release pipeline, the `.pubxml` publish profiles, any
`.publishsettings` file, the Magic Chunks task and the msdeploy PowerShell script from the repository as part of
the change, and note the deletions in `MIGRATION.md`.

## Service principal

Create one principal per pipeline, and one per environment if test and production deploy from different
pipelines or stages — the production principal should never be able to touch anything else.

```bash
litium-cloud service-principal create --name deploy-pipeline --expires 180 -f deploy-pipeline.pem
```

The id is in the output, in the form `service.<name>@cloud`; it is what you pass to `--email` when granting
roles and to `--username` when signing in. The `.pem` file holds the private key followed by the certificate,
unencrypted — anyone holding it can act as the principal. Put it straight into the pipeline's secret store and
delete the local copy. Never commit it. A `.pfx` extension produces a PKCS #12 file instead; the CLI signs in
with either.

Pass `--certificate` as an **absolute path**: the CLI stores the path at sign-in and reads the file again on
every later command, so a relative path or a deleted file makes later commands fail with `Could not connect to
server.` Both assets use an absolute path (`$(certificate.secureFilePath)` / `$RUNNER_TEMP/litium-cloud.pem`).

### Minimum roles

A principal starts with no access and inherits nothing from the user who created it. For creating artifacts and
deploying to one app, the docs list exactly four grants:

| Scope | Role | Command group |
|---|---|---|
| Subscription | `subscription/reader` | `litium-cloud subscription access-control add --email <service-principal-id> --role subscription/reader` |
| Subscription | `artifact/creator` | `litium-cloud subscription access-control add --email <service-principal-id> --role artifact/creator` |
| Environment | `environment/reader` | `litium-cloud environment access-control add --email <service-principal-id> --role environment/reader` |
| App | `appresource/writer` | `litium-cloud app access-control add --app <app-id> --email <service-principal-id> --role appresource/writer` |

`artifact/creator` lets the principal create artifacts and read the ones it created. Run `litium-cloud role list`
for the current ids. Grant at the narrowest scope that works: an app and an environment, not the subscription,
for a production pipeline. The pipeline does **not** need `apps/litium-platform/litium-management` — domain,
`install-app` and force-delete actions stay manual. Recipe: `cicd-service-principal`.

### Certificate expiry and renewal

Certificates are valid for **180 days by default and 365 at most**, so a pipeline that has worked for months
stops signing in with no change of its own. `litium-cloud service-principal list` shows the expiry date of every
principal; put it in the team calendar the day you create it.

```bash
litium-cloud service-principal renew --service-principal <service-principal-id> --expires 180 -f deploy-pipeline.pem
```

Renewing issues a new certificate and **revokes every other active certificate on the principal**, so replace
the pipeline secret in the same sitting or the pipeline breaks immediately. The id and the roles do not change.
`litium-cloud service-principal show --service-principal <id>` lists each certificate with its not-before,
not-after and revoked dates. Add the renewal date to the hand-over notes in `MIGRATION.md`.

## Azure DevOps

`assets/azure-pipelines.yml` is the full template. Prerequisites: the certificate uploaded as a secure file
named `litium-cloud.pem` under **Pipelines > Library > Secure files**, and a variable group `litium-cloud-deploy`
with the secret variables `LitiumNuGetUser`, `LitiumNuGetPassword` and `LitiumServicePrincipal`; `subscription`,
`environment`, `app` and the other plain values are pipeline variables in the YAML. Every step is a `bash` step
on `ubuntu-latest`; secrets reach the scripts through `env:` mappings, ids travel between steps with
`##vso[task.setvariable variable=…]`, and `jq` is preinstalled on the hosted agent. `DownloadSecureFile@1`
(named `certificate`) puts the certificate on the agent and `$(certificate.secureFilePath)` is its absolute path.
Step 2 prepends `~/.dotnet/tools` to `PATH` with `##vso[task.prependpath]`. Azure DevOps sets `TF_BUILD`, so the
CLI runs non-interactively: no browser, no confirmation prompts, `--no-progress` on by default.

Production stage: the docs page shows how to move steps 6 and 7 into `pipelines/deploy-steps.yml` and split the
pipeline into a *Test* stage (build, create the artifact once, deploy to test) and a *Production* `deployment`
job against an Azure DevOps environment with an approval check. Step 4 gets `name: createArtifact` and a second
`setvariable` line with `isOutput=true`; the production stage reads
`$[ stageDependencies.Test.BuildAndDeploy.outputs['createArtifact.artifactId'] ]`, installs the CLI and signs in
again with a production-only service principal (its own variable group and secure file).

## GitHub Actions

`assets/github-actions.yml` is the same flow with repository secrets: `LITIUM_NUGET_USER`,
`LITIUM_NUGET_PASSWORD`, `LITIUM_SP_ID` and `LITIUM_SP_CERTIFICATE` (the full `.pem` content, pasted exactly,
line breaks intact). Secrets are mapped into `env:` of the step that needs them, never written into `run:`
bodies. The workflow writes the certificate to `$RUNNER_TEMP/litium-cloud.pem` with `chmod 600`, passes ids
between steps through `$GITHUB_OUTPUT` and `steps.<id>.outputs`, adds `~/.dotnet/tools` to `$GITHUB_PATH`, and
parses JSON with `jq`, which is preinstalled on GitHub-hosted runners. GitHub Actions sets `CI=true`, which puts
the CLI in the same non-interactive mode.

Production job: the docs page shows a reusable workflow `.github/workflows/deploy-to-litium.yml`
(`workflow_call` with `github_environment`, `subscription`, `environment`, `app` and `artifact_id` inputs) whose
job runs with `environment: ${{ inputs.github_environment }}`, so required reviewers on the *production*
environment gate it and environment secrets supply the production service principal. The caller becomes three
jobs: `build` (steps 1 to 5, `outputs.artifact_id`), `deploy-test` and `deploy-production`, both `uses:` the
reusable workflow with `secrets: inherit`.

The docs page for GitHub Actions carries a note that the example has not been run against a live repository;
say the same to the user and test it against the test environment first.

## Storefront pipeline

Same seven steps, three changes: `artifactType` is `nextjs` (or `nodejs`, `nuxtjs`), the app id is the
storefront app's, and the publish step is removed — the storefront is built in Serverless Cloud from the source,
so `publishPath` points at the storefront folder and `.gitignore` / `.npmignore` / `.litiumcloudignore` keep
`node_modules`, `.git` and build output out of the upload. Optionally keep `npm ci` and `npm run build` before
step 2 so a broken build fails before the upload. The .NET SDK, the NuGet feed and the tool install stay, because
the CLI is a .NET tool. Keep the artifact timeout generous; these artifacts are built after upload.

## Pipeline behaviour to keep in mind

- **Add `-o json` to every command you parse.** Never parse the table output. Null and default values are left
  out of the JSON (`failed` only appears when `true`, `completedAt` only once the job is done).
- **Sign in at the start of every run.** The sign-in is cached on the agent; re-signing in keeps a stale cache
  from surprising you.
- **Exit codes are 0 or 1, and a queued job returns 0.** That is why steps 5 and 7 exist; never treat a
  successful `app deploy` as a successful deployment.
- **`--auto-yes`** is what makes destructive commands work non-interactively. A deployment pipeline does not
  need it; only add it if the user is writing a pipeline and asks for it.
- **Do not print secrets.** Map them through `env:` as both assets do; the scripts never echo them.
- On go-live day the platform app is installed from a manifest, not by the pipeline; the pipeline only has to be
  able to deploy to production afterwards.

## Done when

The pipeline builds a Linux artifact and deploys it to the **test** environment with a service principal and no
manual step, the production target is configured but not yet run, and `MIGRATION.md` records the pipeline
location, the service principal ids, the certificate expiry date and where each secret is stored.
