# Rehearsal and cutover runbook

Derived from https://docs.litium.dev/cloud/serverless/migration/prepare-production.md, `go-live.md` and `after-go-live.md`. Command syntax: `litium-cloud-cli` (recipes `restore`, `install-litium-platform`, `app-lifecycle`, `custom-domain`, `backups`). Every table below uses the same columns; the *Expected duration* column is empty on purpose until the rehearsal has measured it — copy the tables into `MIGRATION.md` and fill it in.

Owners: **partner** (you and the user), **customer** (administrators, DNS owner, business owner), **support** (Litium support, booked at least three working days ahead).

## Gates before you start

- Verified test environment, one manifest per app in version control, production secrets created and listed.
- Production environment created with the production flag, Litium CDN and Litium Insights installed, access restricted.
- Pipeline deploys to test with a service principal.
- Go-live decisions recorded in `MIGRATION.md`: dates, transaction strategy, order prefix, domain path, who moves domains, DNS TTL plan.
- Before any production action: `context show` and `environment show`, and state subscription, environment and production flag.

## The two forks

### LCC vs non-LCC domain switch

| | Already on Fastly through Litium (LCC) | Not on Fastly |
|---|---|---|
| Certificates | Already in the customer's legacy Fastly service; ask support to confirm they cover every domain on the list | Support must add a certificate for every domain to the new Fastly service before go-live |
| DNS | No change | Records change at T-0; TTL lowered to 300 s at least a day before |
| The switch | Move each domain from the legacy Fastly service to the environment's new service (remove from legacy, add to new with a Litium CDN domain app or the CDN `add-domain-name` action), then `add-domain` / `replace-domain` on the Litium platform app and map to the channel | Add every domain to Litium CDN and Litium platform in advance, then make the DNS changes |
| Who | Partner with Fastly access, otherwise support during the window | Customer's DNS owner (partner prepares the exact records; support supplies the DNS target) |
| Rollback | Move the domain back to the legacy service | Revert the DNS records (fast because of the low TTL) |

A domain can exist on only one Fastly service at a time. **Confirm the exact removal and addition order with Litium support** before moving domains yourself; a domain that is on both services, or on neither, is unreachable until fixed.

### Transaction strategy

| Data | Option A: freeze | Option B: continue and migrate |
|---|---|---|
| Back office changes (products, content, campaigns) | Administrators stop at the database freeze; changes after it are redone in the new site | Same; there is no automatic merge |
| Orders | Checkout closed with a feature toggle showing a maintenance message from the database freeze until the domains answer from the new site | Sales continue in the legacy site; orders created after the final backup are exported from the legacy database and imported into the new site after go-live. Use a different order number prefix in the new site |
| Other transactions (accounts, subscriptions, reviews) | Feature off during the window | Migrated manually afterwards |
| Payment callbacks for existing orders | Old webhook URLs mapped by support in the new Fastly service (both options) | Same |

Option B adds manual import work and a data-reconciliation step to *After*; get the customer's explicit choice and write it down.

## Rehearsal (in the production environment)

Do the full runbook once with fresh production data, everything except the domain switch. Purpose: find problems that only production data shows, and measure durations.

1. Ask support for a current production database backup and files backup (without `IdentityServer`), three working days ahead (`references/support-requests.md`).
2. Run the *T-1* and *T-0* tables below up to, but not including, **Switch the domains**. Time every step from the moment the command is issued until its verification passes.
3. Test with production data on the system domain or a temporary custom domain: back office sign-in with production administrator accounts; a test order through every payment and delivery method, with the callback updating the order; one run of every integration and scheduled job with its output checked in Litium Insights; the pages and products the customer cares about most.
4. Record in `MIGRATION.md` under *Rehearsal record*: measured durations per step, every fix made (folded into code or manifests, never applied by hand in the environment), every manual step (each becomes a runbook line), and the artifact ids used.
5. Leave the rehearsal installation in place. On go-live day you uninstall it and reinstall from the final backups, which is exactly the runbook again.

Gate to proceed: the runbook completed within the planned window with no unplanned manual steps, and the webhook mapping and domain plan are confirmed by support.

## Before (prepare production, weeks to days ahead)

| Step | Owner | Earliest start | Expected duration | Verification | Rollback |
|---|---|---|---|---|---|
| Create production environment with the production flag; install Litium CDN and Litium Insights from the test manifests | partner | after test environment verified | | `app list` shows both apps **Running**; `environment show` shows production | n/a |
| Restrict access to production; grant the service principal only the deployment roles | partner | with the environment | | `environment access-control show`, `app access-control show` | remove grants |
| Create every production secret; check in one manifest per app | partner | with the environment | | `environment secret list` lists every `secretRef` used in the manifests | n/a |
| Pipeline deploys to test; production deploy target configured but not run | partner | after secrets | | A pipeline run creates an artifact and deploys to test with no manual step | n/a |
| Rehearsal (section above) | partner + support (backups) | after pipeline | measured | Rehearsal record complete | uninstall and retry |
| Inventory webhooks; send old payment webhook URLs and other Fastly configuration to support | partner | after rehearsal test orders | | Support confirms the mapping for the production environment | n/a |
| Domain plan: full list, LCC or non-LCC path, who moves each, certificate coverage confirmed | partner + customer + support | after inventory | | List with a responsible person per domain in `MIGRATION.md` | n/a |
| Non-LCC only: ask support for certificates and the DNS target; add every domain to Litium CDN and Litium platform | partner + support | as soon as the domain list is final | lead time from support | Domains listed in `app list` (CDN domain apps) and in the back office | delete the CDN domain apps |
| Non-LCC only: lower DNS TTL to 300 s | customer (DNS owner) | at least one day before go-live | | `dig`/`nslookup` shows the new TTL | raise TTL |
| Book support for the go-live window: date, final database backup (who, when, direct upload?), files backup, domain move if support does it, contact channel | partner | three or more working days before | | Written confirmation from support | reschedule |
| Change freeze notice to administrators in writing: media stop (T-1), content/product stop (database freeze), checkout toggle times, which legacy jobs are paused | partner → customer | with the booking | | Customer acknowledges | n/a |

## T-1 (the day before)

| Step | Owner | Earliest start | Expected duration | Verification | Rollback |
|---|---|---|---|---|---|
| Confirm freeze, checkout toggle readiness, support bookings, contact details | partner + customer + support | morning T-1 | | Everyone has the runbook and the time plan | reschedule |
| Uninstall the rehearsal's dependent apps (payment, delivery, storefront, any private app with `appRef` to the platform app), then the Litium platform app. **Show each `app delete` and get a yes.** Leave Litium CDN, Insights, File storage, SFTP, SMTP relay | partner | after confirmation | measured | `app list` shows neither the platform app nor its dependents; manifests in version control | reapply the manifests |
| Customer stops media uploads; take the files backup (or receive it from support), check it has no `IdentityServer` folder, upload as a `storage` artifact with `--file-path` at the folder that contains `media` | customer + partner (or support) | after media stop | measured (longest upload) | `artifact show` reports **Ready**; id written to `MIGRATION.md` | re-upload |

If a Litium CDN domain app references the platform app and the platform delete is refused because of it, delete that domain app first and reinstall it after the platform is back; confirm with Litium support whether domain apps must be recreated when the platform app is reinstalled.

## T-0 (go-live day)

| Step | Owner | Earliest start | Expected duration | Verification | Rollback |
|---|---|---|---|---|---|
| Pause legacy scheduled jobs and integrations; turn on the checkout toggle (if freeze strategy); announce the database freeze | customer + partner | window start | | No legacy job runs; maintenance message visible | resume legacy jobs, toggle off |
| Take the final production database backup as agreed; upload as a `sqlbackup` artifact (or receive the id from support) | support or partner | after the freeze | measured | `artifact show` reports **Ready**; id noted | retake |
| Update `litium-platform.yaml`: `artifact` = latest production code artifact from the pipeline, `sql_backup_file`, `storage_backup_file`; commit | partner | after both ids | | Diff shows only the three ids | revert commit |
| **Apply the Litium platform manifest** (show the command, confirm target with `context show`, get a yes); follow the job with `status logs --follow` | partner | after commit | measured (longest step after the upload) | Job **Completed**; `app show` **Running**; back office opens on the system domain with `/Litium` | fix and re-apply, or `app delete` and reinstall |
| Force-delete the legacy payment and delivery apps (back office, or `uninstall-app` action with `id` and `force=true`; needs `apps/litium-platform/litium-management`) | partner | after the platform is Running | measured | Apps page in the back office shows none of the legacy apps | n/a (they cannot work in the new site) |
| Apply payment, delivery, storefront and other dependent manifests in order; register each payment/delivery app in the back office and upload its configuration file | partner | after force-delete | measured | `app list` all **Running**; each app installed and configured in the back office | `app delete` and re-apply |
| Rebuild search indexes (`rebuild-search-indicies` action or back office **Settings**) | partner | after platform Running | measured | A product search on the site returns results | rerun |
| Smoke test on the system domain: latest legacy orders and content present; start page, category, product and checkout on every channel; test order through the main payment method with callback; no errors in Litium Insights since start | partner + customer | after indexes | measured | All checks pass | stop here; legacy is untouched |
| **Switch the domains** per the fork above; turn the checkout toggle off as soon as the production domains answer from the new site | partner or support (LCC) / customer DNS owner (non-LCC) | after smoke test | measured | Every domain opens the new site over HTTPS; legacy request log drops to zero | move domains back / revert DNS |
| Verify old webhooks: trigger a callback for a legacy order (capture or refund from the provider portal) and check the order updates in the new back office; update URLs in every other external system from the inventory | partner + customer | after the switch | | Old and new URLs both update orders | escalate to support (mapping) |
| Resume scheduled jobs and integrations in the new site; confirm the legacy jobs stay paused | partner + customer | after webhooks | | First run of each integration completes and logs to Insights | pause again |

## After (first days and decommissioning)

| Step | Owner | Earliest start | Expected duration | Verification | Rollback |
|---|---|---|---|---|---|
| Monitor Litium Insights: errors, slow requests, failed jobs; compare order volume with the same weekday before | partner + customer | T-0 evening, then daily | | Error rate and order volume in line with legacy | fix forward |
| Monitor legacy logs for stray traffic (forgotten domain, hard-coded integration URL, unmapped callback); fix at the sender or via the mapping | partner | T+1 | | Legacy receives no traffic apart from your checks | n/a |
| Option B only: export orders and other transactions from the switch window from the legacy database and import into the new site | partner + customer | T+1 | | Every window order exists with the right state; customer confirms the count | n/a |
| Verify every scheduled job and integration has run on schedule; if jobs compete with web traffic, ask support for a worker node (8.16+ production) | partner | T+1 | | Each job has a successful run logged after go-live | n/a |
| Remove the checkout toggle and other temporary switches; deploy through the pipeline. Keep the new order prefix | partner | after stable days | | Pipeline run deployed; toggle gone | n/a |
| Keep a final copy of the legacy database and files backups; confirm nothing else reads from the legacy servers (SFTP not in the inventory); confirm the old webhook mapping lives in the new Fastly service | partner + customer | after sign-off | | Copies downloaded and stored; checks recorded | n/a |
| Ask support to delete the legacy environment (the customer pays for both until then) | partner → support | after sign-off and window data migrated | | Support confirms deletion | none — irreversible |
| Non-LCC only: raise the DNS TTL back | customer | after a few stable days | | `dig` shows the normal TTL | n/a |
| Hand over: manifests and pipeline location, secret list with owners, access setup, SFTP hosts/users/allow lists, webhook inventory, measured durations, links to backups and monitoring guides | partner → customer | with sign-off | | `MIGRATION.md` *Hand-over* section complete | n/a |

## Go / no-go decision points on T-0

Agree these with the customer in advance and write the names of the deciders into the plan. At each point, state the measured time against the plan and the verification result before asking for the decision.

| Point | Question | Go | No-go |
|---|---|---|---|
| After the final database artifact is **Ready** | Is the backup complete and from after the freeze? | Apply the platform manifest | Retake the backup; the freeze continues |
| After the platform job **Completed** | Does the back office open and show the latest legacy orders and content? | Reinstall dependent apps | Read the job log; if the database restore or upgrade failed, `app delete` (with a yes) and re-apply; if it is a code problem, fix forward through the pipeline while the legacy site is still live |
| After the smoke test | Test order placed, callback received, no errors in Insights, search works on every channel? | Switch the domains | Stop: nothing is public. Fix and repeat the smoke test, or reschedule and lift the freeze |
| After the domain switch (first 30 minutes) | Every domain answers from the new site over HTTPS, first real orders complete, no callback errors? | Verify old webhooks, resume integrations | Revert the domains or DNS while no new orders exist (see Rollback rules) |
| End of the window | Integrations resumed and first runs logged, legacy jobs paused, customer informed? | Close the window, start *After* | Keep the freeze on the legacy side until resolved |

## Smoke test in detail

Run it on the system domain (or a temporary custom domain) before any domain is switched, and repeat the first three items on the production domains right after the switch. Record each result in the cutover log.

1. Back office: sign in with a production administrator account; open the order list and check that the most recent legacy orders (from just before the freeze) are present; open a recently edited product and a recently edited page.
2. Storefront, per channel and language: start page, a category with filters, a product page with images from the restored media, search for a known product, add to cart.
3. Checkout: a real test order through the main payment method; confirm the callback updates the order state; if delivery options are provider-driven, check that they load. Cancel or refund the test order afterwards and note that as a callback test.
4. Litium Insights: app logs since the app started with no errors; request log shows the pages you opened.
5. Jobs: trigger one integration run (or wait for the first scheduled run) and check its output, including number and date formats.
6. Mail: trigger one transactional mail (order confirmation or password reset) and confirm it arrives from the right sender.
7. SFTP (if used): the external system connects and sees its folders; a file dropped in a mounted folder is visible to the app under `/app_storage/<name>`.
8. Nothing points at legacy: `app show` on the platform app and a look at the manifests show no legacy host names; the checkout toggle is ready to be turned off.

## Timing worksheet

Copy this into `MIGRATION.md` after the rehearsal and again after go-live.

```text
Step                                   Rehearsal   Go-live   Owner
Files backup taken (legacy side)       __h__m      __h__m    customer/support
Storage artifact upload -> Ready       __h__m      __h__m    partner/support
Rehearsal uninstall (dependents+platform) __h__m   __h__m    partner
Database backup taken (legacy side)    __h__m      __h__m    support/partner
Database artifact upload -> Ready      __h__m      __h__m    partner/support
Platform apply -> Running              __h__m      __h__m    partner
Force-delete legacy apps               __h__m      __h__m    partner
Dependent apps applied and registered  __h__m      __h__m    partner
Search index rebuild                   __h__m      __h__m    partner
Smoke test                             __h__m      __h__m    partner+customer
Domain switch until all domains answer n/a         __h__m    partner/support/customer
Old webhook verification               n/a         __h__m    partner+customer
Total from freeze to traffic           __h__m      __h__m
```

The go-live window must be longer than the rehearsal total plus the domain switch plus a margin for one retry of the longest step.

## Rollback rules

- **Before the domain switch**: nothing is public. Fix and continue, or stop and reschedule; the legacy site is untouched. Do not delete anything in the legacy site during go-live.
- **After the switch, before any new order**: revert the domain move or the DNS change so traffic returns to legacy; resume the legacy integrations; keep the Serverless Cloud installation for analysis.
- **After new orders exist in the new site**: a rollback requires exporting those orders and importing them into the legacy database by hand. Decide with the customer whether to roll back or fix forward, and plan the export before reverting the domains.
- Never roll back by re-applying a manifest with older backup ids: `sql_backup_file` and `storage_backup_file` are create-only and ignored on an existing app.

## What to record in MIGRATION.md

- *Rehearsal record*: date, backup artifact ids, duration per step, fixes, manual steps, test-order results per payment method.
- *Cutover plan*: the filled T-1 and T-0 tables with names and times, the domain list with owners, the support contact channel.
- *Cutover log*: actual times, job ids, artifact ids, deviations, who did what.
- *After*: monitoring notes, window-order reconciliation, decommission confirmation, hand-over checklist.
