# Litium Cloud CLI — Workflow Recipes

Step-by-step guides for common cloud operations. Each recipe shows the exact CLI commands in order.

> Tip: Run `litium-cloud context set --subscription <id> --environment <id>` once before starting to avoid repeating those flags on every command.

---

## `new-environment` — Set up a new Litium environment from scratch

This is the full lifecycle for a fresh environment including CDN, Platform, and Insights.

```bash
# 1. Confirm you are logged in
litium-cloud auth show

# 2. Find your subscription ID
litium-cloud subscription list

# 3. Find available deployment locations
litium-cloud location list

# 4. Create the environment
#    Add --production for a production environment
litium-cloud environment create \
  --name staging \
  --subscription <sub-id> \
  --location <location-id> \
  --set-context

# 5. Set context so subsequent commands don't need --sub/--env
litium-cloud context set --subscription <sub-id> --environment <env-id>

# 6. Install Litium CDN first (must be installed before Litium Platform)
#    See install-cdn-insights workflow below

# 7. Install Litium Platform
#    See install-litium-platform workflow below

# 8. (Optional) Install Litium Insights
#    See install-cdn-insights workflow below
```

---

## `install-cdn-insights` — Install Litium CDN and Insights

CDN **must** be installed before Litium Platform. Without CDN, Platform won't be automatically registered in Fastly.

```bash
# 1. Download the CDN manifest
litium-cloud marketplace manifest --app litium-cdn -f litium-cdn.yaml

# 2. Apply the CDN manifest
litium-cloud apply -f litium-cdn.yaml

# Note the job ID printed and wait for it to finish
litium-cloud status show --job <job-id>

# 3. (Optional) Install Litium Insights for logs and metrics
litium-cloud marketplace manifest --app litium-insights -f litium-insights.yaml
litium-cloud apply -f litium-insights.yaml
litium-cloud status show --job <job-id>
```

---

## `install-litium-platform` — Install the Litium Platform app

Prerequisites: environment created, CDN installed, .NET artifact ready (or skip to use latest Litium version).

```bash
# 1. Download the Litium Platform manifest
litium-cloud marketplace manifest --app litium-platform -f litium-platform.yaml

# 2. (Optional) Include your own .NET artifact
#    a. Create artifact first (see deploy-dotnet workflow)
#    b. Uncomment the "properties" section in litium-platform.yaml
#    c. Set:   - name: artifact
#                value: artifacts/<your-artifact-id>

# 3. (Optional) Include a SQL database backup
#    a. Create/upload SQL backup artifact:
litium-cloud artifact create --file-path ./backup.bak --artifact-type sql-backup --sub <sub-id>
#    b. Uncomment and set in litium-platform.yaml:
#         - name: sql_backup_file
#           value: artifacts/<sql-artifact-id>

# 4. (Optional) Include a storage backup
litium-cloud artifact create --file-path ./storage.zip --artifact-type storage-backup --sub <sub-id>
#    Set in litium-platform.yaml:
#         - name: storage_backup_file
#           value: artifacts/<storage-artifact-id>

# 5. Install the platform
litium-cloud apply -f litium-platform.yaml
litium-cloud status show --job <job-id>

# 6. Find the public URL of the installed platform
litium-cloud app show --app litium
# Look for "public_domain_name" under Exposes
# Default login: admin / Password!

# 7. Verify installed apps
litium-cloud app list
```

**Note:** If no artifact is provided, the latest available Litium version is installed automatically.

---

## `deploy-dotnet` — Build and deploy a .NET artifact (MVC Accelerator)

Use this workflow for continuous deployments after the platform is already installed.

```bash
# Step A: Build the artifact (on developer machine / CI agent)

# 1. Add the cloud manifest generator package to the .NET project (one-time setup)
dotnet add ./Src/Litium.Accelerator.Mvc/Litium.Accelerator.Mvc.csproj \
  package Litium.Cloud.Tools.Targets --prerelease

# 2. Publish the project (adjust framework version as needed)
dotnet publish ./Src/Litium.Accelerator.Mvc/Litium.Accelerator.Mvc.csproj \
  -f net9.0 -c Release -o ./publish -a x64 --os linux

# 3. Create the artifact in Litium Cloud
litium-cloud artifact create \
  --name "Release $(date +%Y-%m-%d)" \
  --file-path ./publish/ \
  --artifact-type dotnet \
  --subscription <sub-id>
# Note the artifact ID from the output

# Step B: Deploy the artifact to the running app

# 4. Deploy to the Litium Platform app (app ID is typically "litium")
litium-cloud app deploy \
  --app litium \
  --artifact <artifact-id>
# Note the job ID from the output

# 5. Monitor deployment
litium-cloud status show --job <job-id>
# Rolling deployment: zero downtime, replicas updated one by one

# 6. Verify
litium-cloud app show --app litium
```

**Important:** The `license.json` file must be included in the published output (inside the artifact). Request a license from Litium if needed.

---

## `deploy-nextjs` — Build and deploy a Next.js storefront

Use after the Litium Storefront app is installed, or for initial installation.

```bash
# Step A: Build the artifact

# 1. Build the Next.js application
npm run build   # or: yarn build

# 2. Create the artifact
litium-cloud artifact create \
  --name "Storefront $(date +%Y-%m-%d)" \
  --file-path ./ \
  --artifact-type nextjs \
  --subscription <sub-id>
# Note the artifact ID

# Step B: Install (first time) or deploy (update)

# First-time installation:
litium-cloud marketplace manifest --app litium-storefront -f litium-storefront.yaml
# Edit litium-storefront.yaml to reference your artifact:
#   - name: artifact
#     value: artifacts/<your-artifact-id>
litium-cloud apply -f litium-storefront.yaml
litium-cloud status show --job <job-id>

# Subsequent deployments (app already installed):
litium-cloud app deploy --app litium-storefront --artifact <artifact-id>
litium-cloud status show --job <job-id>
```

**Tip:** Use `.litiumcloudignore` (similar to `.gitignore`) to exclude files from the artifact (e.g. `node_modules/`, `.git/`).

---

## `cicd-service-principal` — Set up CI/CD with a service principal

Service principals allow pipelines (GitHub Actions, Azure DevOps, etc.) to authenticate without a user account.

```bash
# 1. Create the service principal (run once, on your local machine)
litium-cloud service-principal create \
  --name "GitHub Actions Deploy" \
  --expires 365 \
  --file ./deploy-cert.pem
# Output includes the service principal's email address (e.g. sp-abc123@serviceprincipal.litium.cloud)

# 2. Find the service principal email
litium-cloud service-principal list

# 3. Grant the service principal minimum required permissions:

# subscription/reader (required to read subscription)
litium-cloud subscription access-control add \
  --email <sp-email> \
  --role subscription/reader \
  --sub <sub-id>

# environment/reader (required to read environment)
litium-cloud environment access-control add \
  --email <sp-email> \
  --role environment/reader \
  --sub <sub-id> \
  --env <env-id>

# appresource/writer on the target app (required to deploy)
litium-cloud app access-control add \
  --email <sp-email> \
  --role appresource/writer \
  --sub <sub-id> \
  --env <env-id> \
  --app <app-id>

# artifact/creator (to upload new artifacts)
litium-cloud subscription access-control add \
  --email <sp-email> \
  --role artifact/creator \
  --sub <sub-id>

# 4. Store the certificate securely in your CI system
#    (e.g. GitHub Actions secret, Azure DevOps secure file)

# 5. In your pipeline, log in and deploy:
litium-cloud auth login \
  --service-principal \
  --username <sp-email> \
  --certificate ${{ secrets.LITIUM_CERT }}

litium-cloud artifact create \
  --file-path ./publish/ \
  --artifact-type dotnet \
  --subscription <sub-id>

litium-cloud app deploy \
  --app litium \
  --artifact <artifact-id> \
  --subscription <sub-id> \
  --environment <env-id>
```

**Certificate renewal** (before expiry):
```bash
litium-cloud service-principal renew \
  --service-principal <sp-id> \
  --expires 365 \
  --file ./deploy-cert-new.pem
```

---

## `backups` — Create and restore database/storage backups

### Create a database backup

```bash
# Trigger the backup (context must be set, or pass --sub and --env)
litium-cloud app action --action backup-database --app litium

# Note the job ID and wait for it to finish
litium-cloud status show --job <job-id>

# Find the backup artifact
litium-cloud artifact list --filter "Database backup"
# Note the artifact ID for the backup

# (Optional) Download the backup locally
litium-cloud artifact download --artifact <artifact-id> --file ./backup.bak
```

### Create a storage backup

```bash
# Backup only media-related storage (for migration between environments)
litium-cloud app action --action backup-storage --app litium

# Backup everything in the storage folder
litium-cloud app action --action backup-storage --app litium --property complete=true

litium-cloud status show --job <job-id>

litium-cloud artifact list --filter "Persistent storage backup"
litium-cloud artifact download --artifact <artifact-id> --file ./storage-backup.zip
```

### Restore a backup

Backups are restored during Litium Platform installation by referencing the artifact in the manifest YAML:

```yaml
# In litium-platform.yaml
spec:
  properties:
    - name: sql_backup_file
      value: artifacts/<sql-backup-artifact-id>
    - name: storage_backup_file
      value: artifacts/<storage-backup-artifact-id>
```

**Warning:** You cannot restore a backup to an already-installed Litium App. You must delete the app and reinstall it with the backup artifact referenced in the manifest.

---

## `access-control` — Manage user and group access

### Grant a user access to an environment

```bash
# Give a user read access to the subscription (prerequisite for most operations)
litium-cloud subscription access-control add \
  --email user@company.com \
  --role system/reader \
  --sub <sub-id>

# Give the user access to a specific environment
litium-cloud environment access-control add \
  --email user@company.com \
  --role system/contributor \
  --sub <sub-id> \
  --env <env-id>
```

### Create a group and assign access

```bash
# Create a group
litium-cloud group create --name "QA Team" --description "Quality assurance"
# Note the group ID

# Add members
litium-cloud group member add --group <group-id> --email qa1@company.com
litium-cloud group member add --group <group-id> --email qa2@company.com

# Find the group's email address
litium-cloud group show --group <group-id>
# Look for the email field (e.g. qateam@groups.litium.cloud)

# Grant the group access
litium-cloud environment access-control add \
  --email <group-email> \
  --role environment/reader \
  --sub <sub-id> \
  --env <env-id>
```

### Grant Insights access

Users need `system/reader` on the subscription and `system/contributor` on the Insights app.

```bash
litium-cloud subscription access-control add \
  --email user@company.com \
  --role system/reader \
  --sub <sub-id>

litium-cloud app access-control add \
  --email user@company.com \
  --role system/contributor \
  --app litium-insights \
  --sub <sub-id> \
  --env <env-id>
```

### View access

```bash
litium-cloud subscription access-control show --sub <sub-id>
litium-cloud environment access-control show  --sub <sub-id> --env <env-id>
litium-cloud app access-control show          --app <app-id>
```

---

## `copy-environment` — Copy one environment to another (e.g. test → production)

This workflow replicates all apps and settings from a source environment to a new target environment. Useful for promoting test to production, or creating nightly test refreshes.

```bash
# Prerequisites:
# - Source environment with apps installed
# - Target environment already created (empty)
# - Both environments in the same subscription

# STEP 1: Export YAML for every app in the source environment

# 1a. List all apps in source
litium-cloud app list --env <source-env-id> --sub <sub-id>

# 1b. Export YAML for each app
litium-cloud app show -o manifest --app litium        --env <source-env-id> --sub <sub-id> > litium-platform.yaml
litium-cloud app show -o manifest --app litium-cdn    --env <source-env-id> --sub <sub-id> > litium-cdn.yaml
litium-cloud app show -o manifest --app litium-insights --env <source-env-id> --sub <sub-id> > litium-insights.yaml
# Repeat for each app

# 1c. Create database backup from source
litium-cloud app action --action backup-database --app litium --env <source-env-id> --sub <sub-id>
litium-cloud status show --job <job-id>
litium-cloud artifact list --filter "Database backup"
# Note the artifact ID, then add to litium-platform.yaml:
#   - name: sql_backup_file
#     value: artifacts/<artifact-id>

# 1d. Create storage backup from source
litium-cloud app action --action backup-storage --app litium --env <source-env-id> --sub <sub-id>
litium-cloud status show --job <job-id>
litium-cloud artifact list --filter "Persistent storage backup"
# Note the artifact ID, then add to litium-platform.yaml:
#   - name: storage_backup_file
#     value: artifacts/<artifact-id>

# STEP 2: Install apps into the target environment in order

# 2a. CDN first
litium-cloud apply -f litium-cdn.yaml --env <target-env-id> --sub <sub-id>
litium-cloud status show --job <job-id>

# 2b. Insights
litium-cloud apply -f litium-insights.yaml --env <target-env-id> --sub <sub-id>
litium-cloud status show --job <job-id>

# 2c. Litium Platform (with database and storage backups)
litium-cloud apply -f litium-platform.yaml --env <target-env-id> --sub <sub-id>
litium-cloud status show --job <job-id>

# 2d. Remaining apps (storefront, payment, delivery, etc.)
litium-cloud apply -f <other-app>.yaml --env <target-env-id> --sub <sub-id>
# Wait for each job before applying the next

# STEP 3: Post-installation

# 3a. Verify app URLs
litium-cloud app show --app litium --env <target-env-id> --sub <sub-id>

# 3b. Payment and delivery apps must be reinstalled in Litium Backoffice
#     (uninstall → reinstall in Settings > Apps in the Backoffice UI)
#     OR via CLI:
litium-cloud app action --action uninstall-app --app litium --property app=<payment-app-id> --env <target-env-id>
litium-cloud app action --action install-app   --app litium --property app=<payment-app-id> --env <target-env-id>
litium-cloud app action --action configure-app --app litium --property app=<payment-app-id> --env <target-env-id>

# 3c. Rebuild search indices
litium-cloud app action --action rebuild-search-indicies --app litium --env <target-env-id>

# 3d. Update domain names in Backoffice channels to match the new environment's domains
litium-cloud app action --action remove-domain --app litium --property domain=<old-domain>
litium-cloud app action --action add-domain    --app litium --property domain=<new-domain>

# STEP 4: Verify
litium-cloud app list --env <target-env-id> --sub <sub-id>
```

**Tip:** This workflow is also useful as a nightly automation: create a cron job that exports backups from production and restores them to the test environment overnight using a service principal.
