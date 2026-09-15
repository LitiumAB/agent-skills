# Support requests

Litium support handles the parts of a migration that are not self-service. Open a case with Litium support via https://docs.litium.com/resources/support (the channel the docs point to; the Portal's **Issue tracking** button is for reporting problems, not for booking help). Scheduled help — backups on a date, uploading backups to a subscription, go-live assistance — needs **at least three working days** of lead time. Group requests where you can; one case per migration keeps the history in one place.

Rules for the agent: produce the request text, fill in what you know from `MIGRATION.md`, mark the rest `<...>` for the user, and stop. Never send it yourself. Log every request and answer under *Support requests* in `MIGRATION.md`. Never include secrets, passwords or certificate contents in a request.

## 1. Subscription access

Required: customer name, subscription id (if known), the Litium Account e-mail addresses to grant, the role wanted (`system/owner` for the partner lead, `system/contributor` for developers). Lead time: the docs state none for access requests; allow the usual three working days. You get back: confirmation; verify with `litium-cloud subscription list` and `subscription access-control show`. After the first owner exists, grant further access yourself (`litium-cloud-cli` recipe `access-control`).

```text
Subject: Serverless Cloud subscription access – <customer>

Hi,
We are migrating <customer>'s Litium 8 site from legacy cloud to Serverless Cloud.
Please grant access to the customer's Serverless Cloud subscription (<subscription-id or "please tell us the id">):
- <name>, <litium-account e-mail>, role system/owner
- <name>, <litium-account e-mail>, role system/contributor
The accounts already exist as Litium Accounts. Thanks,
<name, partner company, phone>
```

## 2. Fastly access and a test domain

Required: subscription id, environment id, whether the customer's legacy site is on Fastly through Litium today, the test domain and its DNS owner. Lead time: at least three working days if a certificate has to be added (custom domains guide). You get back: Fastly access for the named accounts (if granted), confirmation that a certificate covers the test domain or that one has been added, and the DNS target. Verify by opening the test domain over HTTPS after you have added it to Litium CDN and the platform app (`litium-cloud-cli` recipe `custom-domain`).

```text
Subject: Fastly access and test domain – <customer>, environment <environment-id>

Hi,
For the migration of <customer> we have installed Litium CDN in subscription <subscription-id>, environment <environment-id>.
1. Please give <e-mail(s)> access to the environment's Fastly service so we can manage domains ourselves.
   The legacy site <is / is not> on Fastly through Litium today.
2. We will use <test.customer-domain> as a test domain. Please confirm a certificate covers it (or add one) and tell us the DNS target to point it at.
Thanks, <name, partner company, phone>
```

## 3. Legacy database and files backups

Required: legacy site name and environment (production), target subscription id, whether support should upload the backups directly as artifacts, the date and time window, and that the files backup must exclude the `IdentityServer` folder. Lead time: at least three working days for a scheduled date. You get back: a `.bak` file and a files archive (the folder containing `media`), or artifact ids if support uploads them. Verify: `artifact show` reports **Ready** for each id; the files archive contains `media` and no `IdentityServer`. Ask for fresh backups again before the rehearsal and for the final ones at go-live; an artifact that no app references is deleted 14 days after creation if it was never used, or 7 days after the last app stopped using it (backups overview), so do not upload a backup long before you plan to use it.

```text
Subject: Legacy backups for Serverless Cloud migration – <customer>

Hi,
Please take a backup of <customer>'s legacy production site for the migration to Serverless Cloud:
- Database backup as a .bak file.
- Files backup of the shared storage (the folder that contains the media folder), excluding the IdentityServer folder.
- When: <date and time window>, at least three working days from now. Purpose: <test environment / rehearsal / final go-live>.
- Target: subscription <subscription-id>. Please <upload both directly as artifacts to the subscription and send us the artifact ids / provide download links>.
Thanks, <name, partner company, phone>
```

## 4. Go-live day help

Required: go-live date and window, subscription and environment ids, what support should do (final database backup and direct upload, files backup upload, domain move if the partner lacks Fastly access), the domain list with the order of the move, the old payment webhook URLs with their new targets, any other Fastly configuration to carry over (dictionaries, redirects), and the contact channel. Lead time: at least three working days; send the webhook list as soon as the rehearsal test orders are done. You get back: written confirmation of the booking, the webhook mapping and the Fastly configuration for the production environment. Verify on T-0: callbacks to old URLs update orders in the new site; every domain opens the new site.

```text
Subject: Go-live booking – <customer>, <date> <time window>

Hi,
We go live with <customer> on Serverless Cloud on <date>, window <start–end, time zone>.
Subscription <subscription-id>, production environment <environment-id>.
Please assist with:
1. Final production database backup at <time> and direct upload as a sqlbackup artifact to the subscription (send us the artifact id).
2. Files backup (excluding IdentityServer) on <date T-1>, uploaded as a storage artifact (send us the id).
3. <Domain move: move the following domains from the legacy Fastly service to the environment's service at <time>, in this order: ... / Not needed, we have Fastly access.>
4. Old payment webhook URLs: please configure the new Fastly service so requests to these old URLs reach the new payment apps after go-live, and keep the mapping in place after the legacy environment is deleted:
   - <old URL> → <new app public domain from app show>
   - ...
5. Carry over other Fastly configuration from the legacy service: <dictionaries, redirects, none>.
Contact during the window: <name, phone, channel>. Please confirm.
Thanks, <name, partner company>
```

## 5. Certificates for domains not in Fastly (non-LCC path)

Required: subscription and environment ids, the full domain list (including redirect domains and the back office domain), the target app for each (Litium platform or storefront), where DNS is managed. Lead time: at least three working days (custom domains guide); send the list as soon as it is final, before the rehearsal. You get back: confirmation that certificates exist for each domain and the DNS records to set at go-live. Verify: after DNS points at the CDN, every domain opens over HTTPS without warnings.

```text
Subject: Certificates for custom domains – <customer>, environment <environment-id>

Hi,
<customer>'s legacy site is not on Fastly today. For subscription <subscription-id>, environment <environment-id>, please add certificates to the environment's Fastly service for:
- <www.customer.com> → Litium platform app <app-id>
- <customer.com> (redirect) → <app-id>
- <shop.customer.com> → storefront app <app-id>
and tell us the DNS records to point these domains at. Planned go-live: <date>. We will lower the DNS TTL to 300 seconds the day before.
Thanks, <name, partner company>
```

## 6. Delete the legacy environment (after go-live)

Required: legacy site name, confirmation that the customer has signed off, that transactions from the switch window are migrated, that final copies of the legacy backups are stored, and that nothing else reads from the legacy servers. Lead time: normal case handling. You get back: confirmation of deletion; the customer stops paying for the legacy environment. This is irreversible: get the customer's written go-ahead first, and make sure the old webhook mapping lives in the new Fastly service before you ask.

```text
Subject: Decommission legacy environment – <customer>

Hi,
<customer> has been live on Serverless Cloud (subscription <subscription-id>, environment <environment-id>) since <date>. The customer has signed off, transactions from the switch window are migrated, and we have stored final copies of the legacy database and files backups.
Please delete the legacy environment <legacy site name / servers>. Please confirm that the old payment webhook mapping in the new Fastly service stays in place.
Thanks, <name, partner company>
```

## What to log in MIGRATION.md

For each request: date sent, type (1–6), what was asked, date answered, what was received (artifact ids, confirmations, DNS targets), and any open point. That log is part of the hand-over.
