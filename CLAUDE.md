# Vault — Salesforce Storage Archival SaaS

## What This Product Is

Multi-tenant SaaS that archives Salesforce email bodies and file attachments
to Cloudflare R2, reducing Salesforce storage costs for mid-market companies.

Customers: Salesforce admins at companies with 50–1,000 SF users on Service
Cloud or Sales Cloud who are hitting storage limits or paying overages.

Core value: "You're paying Salesforce $500/month in storage overages.
Vault costs $149/month. ROI is immediate on day one."

---

## Monorepo Structure

```
vault/
  apps/
    web/              ← Next.js 15 App Router (marketing + dashboard)
    worker/           ← Hono on Cloudflare Workers (API + Workflows + Queue)
  packages/
    db/               ← Drizzle ORM schema + migrations for D1
    types/            ← shared TypeScript interfaces and Zod schemas
    sf-client/        ← Salesforce API client (JWT, Bulk, Composite, Metadata)
  CLAUDE.md
  DECISIONS.md
  pnpm-workspace.yaml
  tsconfig.base.json
  package.json
```

---

## Tech Stack (non-negotiable — do not suggest alternatives)

| Layer | Choice |
|---|---|
| Frontend | Next.js 15 App Router, Tailwind CSS v4, TypeScript strict |
| Deployment | OpenNext + Cloudflare Pages |
| API | Hono on Cloudflare Workers |
| Database | Cloudflare D1 with Drizzle ORM (SQLite dialect) |
| Object Storage | Cloudflare R2 |
| Queue | Cloudflare Queues |
| Durable Execution | Cloudflare Workflows |
| Cron | Cloudflare Cron Triggers |
| Auth (platform) | Lucia v3 — email/password + Google OAuth |
| Billing | Stripe metered usage |
| Package Manager | pnpm workspaces |
| Language | TypeScript everywhere, strict mode, no `any` |
| Validation | Zod for all I/O — API bodies, SF API responses, env vars |
| Testing | Vitest + Miniflare (CF bindings) + MSW (SF API mocks) |

---

## Code Rules (enforce on every file written)

- Arrow functions always. No `function` keyword except module exports.
- No `any`. No `as unknown as X`. Proper types or fix the root cause.
- All async functions have explicit return type annotations.
- Zod validates every external boundary: HTTP request bodies, SF API
  responses, environment variables at startup.
- Drizzle for all D1 queries. No raw SQL strings anywhere.
- Error classes: never throw strings. Typed Error subclasses only.
- Imports: no barrel re-exports that cause circular deps.
- File naming: kebab-case for files, PascalCase for classes/types.
- Every new file: compile check with `tsc --noEmit` before moving on.

---

## Environment Variables

```bash
# Cloudflare (set via wrangler secret put or .dev.vars locally)
CLOUDFLARE_ACCOUNT_ID=
CLOUDFLARE_API_TOKEN=

# Salesforce Connected App (ONE keypair for entire platform, not per client)
SF_CONSUMER_KEY=          # Connected App consumer key
SF_PRIVATE_KEY=           # RSA-2048 private key PEM (for JWT Bearer signing)
# Note: public cert (.crt) is uploaded to Connected App — not stored here

# Platform Auth
LUCIA_SECRET=             # 32+ char random string for session signing

# Google OAuth (for platform login, not SF)
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
GOOGLE_REDIRECT_URI=

# Stripe
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=
STRIPE_PRICE_ID_STARTER=
STRIPE_PRICE_ID_GROWTH=
STRIPE_PRICE_ID_ENTERPRISE=

# Internal
INTERNAL_API_SECRET=      # Worker-to-Worker authenticated calls
BASE_URL=                 # https://vault.com in prod, http://localhost:3000 dev
```

---

## Cloudflare Worker Bindings (wrangler.toml)

```toml
[[d1_databases]]
binding = "DB"
database_name = "vault-db"

[[r2_buckets]]
binding = "STORAGE"
bucket_name = "vault-data"

[[queues.producers]]
binding = "BATCH_QUEUE"
queue = "vault-batch-jobs"

[[queues.consumers]]
queue = "vault-batch-jobs"
max_batch_size = 1
max_concurrency = 10
max_retries = 3
dead_letter_queue = "vault-dlq"

[[workflows]]
binding = "ARCHIVE_WORKFLOW"
name = "archive-workflow"
class_name = "ArchiveWorkflow"

[[workflows]]
binding = "RESTORE_WORKFLOW"
name = "restore-workflow"
class_name = "RestoreWorkflow"

[triggers]
crons = [
  "0 * * * *",    # every hour: process scheduled jobs
  "30 * * * *"    # every hour +30min: report billing events to Stripe
]
```

---

## D1 Database Schema

Full schema lives in `packages/db/schema.ts`. Tables:

### users
```
id TEXT PK (uuid)
email TEXT UNIQUE NOT NULL
name TEXT
passwordHash TEXT          -- null if Google OAuth only
googleId TEXT UNIQUE
role TEXT NOT NULL DEFAULT 'user'  -- 'superadmin' | 'user'
createdAt INTEGER NOT NULL
```

### tenants
```
id TEXT PK
slug TEXT UNIQUE NOT NULL      -- globally unique URL-safe name e.g. "acme-corp"
displayName TEXT NOT NULL
ownerUserId TEXT FK users.id
stripeCustomerId TEXT
stripeSubscriptionId TEXT
plan TEXT DEFAULT 'starter'    -- 'starter' | 'growth' | 'enterprise'
createdAt INTEGER NOT NULL
```

### tenantMembers
```
tenantId TEXT FK tenants.id
userId TEXT FK users.id
role TEXT DEFAULT 'member'     -- 'owner' | 'admin' | 'member'
invitedBy TEXT FK users.id
joinedAt INTEGER NOT NULL
PRIMARY KEY (tenantId, userId)
```

### orgRequests
```
id TEXT PK
requestedBy TEXT FK users.id
tenantId TEXT                  -- null until approved
slugRequested TEXT NOT NULL
sfType TEXT NOT NULL           -- 'production' | 'sandbox'
answersJson TEXT               -- onboarding question answers
status TEXT DEFAULT 'pending'  -- 'pending' | 'approved' | 'rejected'
reviewedBy TEXT FK users.id
createdAt INTEGER NOT NULL
reviewedAt INTEGER
```

### sfOrgs
```
id TEXT PK
tenantId TEXT FK tenants.id
sfOrgId TEXT UNIQUE NOT NULL   -- 18-char Salesforce Org ID
sfInstanceUrl TEXT NOT NULL    -- https://client.my.salesforce.com
sfType TEXT NOT NULL           -- 'production' | 'sandbox'
label TEXT NOT NULL            -- "Production", "Dev Sandbox"
setupComplete INTEGER DEFAULT 0    -- 1 = custom fields deployed
integrationUsername TEXT           -- vault-integration@client.com
connectedBy TEXT FK users.id
createdAt INTEGER NOT NULL
```

### sfTokens
```
orgId TEXT PK FK sfOrgs.id
tokenType TEXT NOT NULL        -- 'admin_setup' | 'integration_cache'
accessTokenEnc TEXT NOT NULL   -- AES-256 encrypted
refreshTokenEnc TEXT           -- only for admin_setup type
sfUserId TEXT NOT NULL
expiresAt INTEGER NOT NULL
updatedAt INTEGER NOT NULL
```
Note: integration jobs use JWT Bearer — no stored refresh token.
accessTokenEnc is a short-term cache of the JWT-obtained access token.

### jobs
```
id TEXT PK
orgId TEXT FK sfOrgs.id
tenantId TEXT NOT NULL
type TEXT NOT NULL             -- 'archive' | 'restore'
sfObjectType TEXT NOT NULL     -- 'Case' | 'Opportunity'
filterJson TEXT NOT NULL       -- { status, dateRange, recordType, ids[] }
status TEXT DEFAULT 'queued'   -- 'queued'|'running'|'done'|'failed'|'cancelled'
totalRecords INTEGER DEFAULT 0
processedRecords INTEGER DEFAULT 0
failedRecords INTEGER DEFAULT 0
triggeredByUserId TEXT FK users.id
triggeredBySfUserId TEXT       -- SF user ID if triggered from LWC
cfWorkflowId TEXT              -- Cloudflare Workflow execution ID
createdAt INTEGER NOT NULL
startedAt INTEGER
completedAt INTEGER
```

### jobRecords
```
id TEXT PK
jobId TEXT FK jobs.id
sfRecordId TEXT NOT NULL       -- Case ID or Opportunity ID
status TEXT DEFAULT 'pending'  -- 'pending'|'processing'|'archived'|'restored'|'failed'
emailsArchived INTEGER DEFAULT 0
attachmentsArchived INTEGER DEFAULT 0
feedItemsArchived INTEGER DEFAULT 0
storageFreedBytes INTEGER DEFAULT 0
r2Prefix TEXT                  -- /{tenantId}/Case/{caseId}/
checksum TEXT
sfFieldUpdateError TEXT        -- if validation rule blocked field update
error TEXT
attempts INTEGER DEFAULT 0
updatedAt INTEGER NOT NULL
UNIQUE (jobId, sfRecordId)
```

### auditLog
```
id TEXT PK
tenantId TEXT NOT NULL
orgId TEXT NOT NULL
jobId TEXT
action TEXT NOT NULL           -- 'archive'|'restore'|'view'|'connect'|'setup'|'delete'
actorUserId TEXT               -- platform user
actorSfUserId TEXT             -- SF user (if from LWC)
recordCount INTEGER
metaJson TEXT
createdAt INTEGER NOT NULL
```

### billingEvents
```
id TEXT PK
tenantId TEXT NOT NULL
orgId TEXT NOT NULL
eventType TEXT NOT NULL        -- 'archive' | 'restore'
recordCount INTEGER NOT NULL
stripeReported INTEGER DEFAULT 0
createdAt INTEGER NOT NULL
```

### schedules
```
id TEXT PK
orgId TEXT FK sfOrgs.id
tenantId TEXT NOT NULL
sfObjectType TEXT NOT NULL
filterJson TEXT NOT NULL
cronExpression TEXT NOT NULL
enabled INTEGER DEFAULT 1
lastRunAt INTEGER
lastJobId TEXT
createdAt INTEGER NOT NULL
```

### sfCache
```
key TEXT PK                    -- "{orgId}:{cacheType}"
dataJson TEXT NOT NULL
fetchedAt INTEGER NOT NULL
ttlSeconds INTEGER NOT NULL    -- storage_analysis: 3600, record_types: 86400
```

Indexes:
```sql
CREATE INDEX idx_jobs_org_status ON jobs(orgId, status);
CREATE INDEX idx_job_records_job_status ON jobRecords(jobId, status);
CREATE INDEX idx_audit_tenant_created ON auditLog(tenantId, createdAt);
CREATE INDEX idx_billing_unreported ON billingEvents(stripeReported, createdAt);
CREATE INDEX idx_schedules_enabled ON schedules(enabled, orgId);
```

---

## Salesforce Authentication

### Two separate auth flows — never mix them up:

**1. Admin Setup Flow (Web Server OAuth — one time per org)**
- Purpose: admin connects org interactively, proves authorization
- Used for: Metadata API (deploy custom fields, button, CSP trusted site)
- Result: access_token + refresh_token stored encrypted in sfTokens
  with tokenType = 'admin_setup'
- Who logs in: the SF admin using their own account
- When used: onboarding setup only, not for archive/restore jobs

**2. Integration Job Flow (JWT Bearer — all background processing)**
- Purpose: server-to-server, no human interaction
- Used for: all archive/restore/delete API calls
- Requires: client creates integration user, pre-authorizes via
  App Manager → Connected App → Manage → Edit Policies →
  Permitted Users: "Admin approved users are pre-authorized" →
  add integration user's profile or permission set
- How it works:
  ```
  1. Build JWT: { iss: SF_CONSUMER_KEY, sub: integrationUsername,
                  aud: login/test.salesforce.com, exp: now+180s }
  2. Sign with SF_PRIVATE_KEY (RS256) using jose library
  3. POST to {instanceUrl}/services/oauth2/token
     grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer
     assertion={signedJwt}
  4. Receive access_token (no refresh token, expires ~2hr)
  5. Cache in D1 sfTokens with expiresAt = now + 90min
  ```
- Sandbox JWT audience: https://test.salesforce.com
- Production JWT audience: https://login.salesforce.com

JWT token function signature (packages/sf-client/jwt.ts):
```typescript
export const getSFAccessToken = async (
  org: SFOrg,
  env: Env
): Promise<{ accessToken: string; instanceUrl: string }>
```

---

## Salesforce API Usage

| Operation | API | Reason |
|---|---|---|
| Fetch IDs for filter (< 2k records) | REST SOQL | Synchronous, fast |
| Fetch IDs for filter (> 2k records) | Bulk API 2.0 Query | Async, no timeout |
| Fetch EmailMessage bodies (any count) | Bulk API 2.0 Query | Returns HtmlBody in CSV |
| Fetch ContentDocumentLinks | Bulk API 2.0 Query | Subquery, no ID list needed |
| Fetch related data for 50 known IDs | Composite API | 5 subrequests = 1 API unit |
| Stream attachment binary | REST GET streaming | Only option for binaries |
| Delete 200+ records | Bulk API 2.0 Delete | One job, unlimited count |
| Delete < 200 records | Composite DELETE | Batches of 25 |
| Update custom fields on parent records | Bulk API 2.0 Update | One job, all records |
| Deploy custom fields/button | Metadata API | Configuration changes |

Bulk API 2.0 base: `{instanceUrl}/services/data/v66.0/jobs/query`
Composite API base: `{instanceUrl}/services/data/v66.0/composite`
Metadata deploy: `{instanceUrl}/services/data/v66.0/metadata/deployRequest`

SOQL subquery pattern (avoids passing 10k IDs in WHERE IN):
```sql
SELECT Id, HtmlBody, TextBody, ...
FROM EmailMessage
WHERE ParentId IN (
  SELECT Id FROM Case WHERE Status = 'Closed' AND ClosedDate < 2024-01-01T00:00:00Z
)
```
Use this for all Phase 1 Bulk queries. Never build WHERE IN ({id1},{id2},...) lists.

---

## Archive Job — Complete Flow

### Phase 1 — Bulk Resolve (CoordinatorWorkflow)

Submit 5 Bulk API 2.0 Query jobs simultaneously (Promise.all):

Job A — Case IDs + metadata:
  SELECT Id, CaseNumber, Subject, Status, OwnerId, CreatedDate
  FROM Case WHERE {filters}

Job B — EmailMessages (full fields including HtmlBody):
  SELECT Id, ParentId, HtmlBody, TextBody, Headers, Subject,
         FromName, FromAddress, ToAddress, CcAddress, BccAddress,
         MessageDate, CreatedDate, CreatedById, LastModifiedDate,
         Incoming, HasAttachment, ThreadIdentifier,
         IsExternallyVisible, Status, MessageIdentifier
  FROM EmailMessage
  WHERE ParentId IN (SELECT Id FROM Case WHERE {filters})

Job C — ContentDocumentLinks:
  SELECT Id, ContentDocumentId, LinkedEntityId
  FROM ContentDocumentLink
  WHERE LinkedEntityId IN (SELECT Id FROM Case WHERE {filters})

Job D — ContentDocuments:
  SELECT Id, LatestPublishedVersionId, Title, FileExtension, ContentSize
  FROM ContentDocument
  WHERE Id IN (
    SELECT ContentDocumentId FROM ContentDocumentLink
    WHERE LinkedEntityId IN (SELECT Id FROM Case WHERE {filters})
  )

Job E — FeedItems:
  SELECT Id, ParentId, Body, CreatedDate, CreatedById, Type, IsRichText
  FROM FeedItem
  WHERE ParentId IN (SELECT Id FROM Case WHERE {filters})

Poll all 5 with step.sleep('poll-bulk', '15 seconds') loop.

Stream each result CSV directly SF → R2 (zero Worker memory):
```typescript
const sfResp = await fetch(resultsUrl, { headers: { Authorization: `Bearer ${token}` } })
await env.STORAGE.put(`${tenantId}/phase1/${jobId}/email_messages.csv`, sfResp.body)
```

R2 paths for phase1:
  /{tenantId}/phase1/{jobId}/case_ids.csv
  /{tenantId}/phase1/{jobId}/email_messages.csv
  /{tenantId}/phase1/{jobId}/content_links.csv
  /{tenantId}/phase1/{jobId}/content_documents.csv
  /{tenantId}/phase1/{jobId}/feed_items.csv

Parse case_ids.csv (stream). INSERT OR IGNORE into D1 jobRecords.
Update D1 jobs.totalRecords = count.

### Phase 2 — Queue Fan-out (CoordinatorWorkflow + BatchWorker)

CoordinatorWorkflow:
  Read case_ids.csv from R2 (stream, row by row).
  Chunk into groups of 50.
  env.BATCH_QUEUE.sendBatch(chunks)
  Monitor: step.sleep every 60s, check D1 pending count = 0.

BatchWorker (Queue consumer, 15 min wall time):
  1. Idempotency: if all caseIds already 'archived' in D1 → ack + return.
  2. Get JWT token.
  3. Stream-parse R2 CSVs, filter rows for these 50 caseIds. Zero SF calls.
  4. Write email JSONs to R2 (Promise.all max 10 concurrent).
  5. Write feed JSON, Case snapshots to R2.
  6. Stream attachment binaries SF → R2:
     const sfResp = await fetch(versionDataUrl, { headers })
     await env.STORAGE.put(binaryKey, sfResp.body)  // ReadableStream, zero memory
     Max 5 concurrent streams.
  7. UPDATE D1 jobRecords status='archived', counts.
  8. message.ack()

### Phase 3 — Bulk Delete + Update (CoordinatorWorkflow)

Read email IDs from phase1 CSV → Bulk API 2.0 DELETE EmailMessages.
Read ContentDocumentIds from phase1 CSV → Bulk API 2.0 DELETE ContentDocuments.
Read archived Case IDs from D1 → Bulk API 2.0 UPDATE Cases (set custom fields).
Submit all 3 simultaneously. Poll until complete. allOrNone: false.

Finalize: update D1 jobs status='done', write billingEvents + auditLog,
delete phase1 CSVs from R2, send email notification.

---

## R2 Storage Layout

```
/{tenantId}/
  /phase1/{jobId}/          ← TEMP — deleted after Phase 3
    case_ids.csv
    email_messages.csv      ← full fields including HtmlBody
    content_links.csv
    content_documents.csv
    feed_items.csv

  /Case/{caseId}/           ← PERMANENT
    /emails/{emailId}.json
    /attachments/{cvId}             ← raw binary
    /attachments/{cvId}.meta.json   ← { filename, mimeType, size, cvId, docId }
    /feed/items.json
    /snapshot.json                  ← { CaseNumber, Subject, Status, OwnerId }

  /Opportunity/{oppId}/     ← same structure
```

---

## Restore Flow

1. Snapshot Case.Status via REST GET (before touching anything).
2. Bulk API 2.0 INSERT EmailMessages (all original fields from R2 JSON).
   Requires "Set Audit Fields upon Record Creation" org setting.
3. Stream attachment binaries R2 → POST to SF as new ContentVersion.
   POST ContentDocumentLink to relink each file.
4. Check Case.Status again. If changed (Flow side effect): PATCH back to original.
5. Bulk API 2.0 UPDATE Cases: clear vault__ custom fields.
6. Update D1 jobRecords status='restored'. Write auditLog + billingEvents.

---

## Salesforce Custom Fields (deployed via Metadata API on org setup)

On Case and Opportunity:
- vault__Is_Archived__c      Checkbox, default false
- vault__Archival_Status__c  Picklist: Pending | Archived | Failed
- vault__Archived_At__c      DateTime
- vault__Archive_Job_Id__c   Text(36)

Also deployed:
- Custom Button on Case: "View Archived Content"
  URL: https://vault.com/sf-view/{!Case.Id}?token={!$Api.Session_ID}
  Target: New Window
- CSP Trusted Site: VaultApp → https://vault.com

---

## Onboarding Flow (5 steps)

1. Web Server OAuth → installs Connected App, runs Metadata API deploy
2. Admin enables "Set Audit Fields upon Record Creation" in SF settings
3. Admin creates integration user (Profile: Minimum Access – API Only)
4. Admin pre-authorizes user: App Manager → Vault → Manage → Edit Policies
   → Permitted Users: Admin approved → add integration user's profile
5. Admin enters integration username in dashboard
   → We test JWT Bearer → ✓ connected

---

## API Routes (Hono)

```
POST /api/auth/signup
POST /api/auth/login
GET  /api/auth/google
GET  /api/auth/google/callback
POST /api/auth/logout

GET  /api/sf/connect              ← returns SF OAuth URL
GET  /api/sf/callback             ← OAuth callback + Metadata API deploy
POST /api/sf/setup-integration-user  ← test JWT + store username

GET  /api/orgs                    ← list orgs for tenant
POST /api/orgs/request            ← submit org request
GET  /api/storage                 ← storage breakdown (1hr D1 cache)

POST /api/jobs                    ← create + trigger ArchiveWorkflow
GET  /api/jobs                    ← list jobs (query: orgId, status)
GET  /api/jobs/:id                ← status + progress from D1
POST /api/jobs/preview            ← COUNT query, return estimate

POST /api/restore                 ← trigger RestoreWorkflow
GET  /api/restore/search          ← search D1 jobRecords (archived)

GET  /api/view/:caseId            ← render archived emails as HTML
                                     validates SF session token
                                     reads R2, builds HTML thread
                                     attachment links = 30s presigned URLs

GET  /api/schedule
POST /api/schedule
PATCH /api/schedule/:id
DELETE /api/schedule/:id

GET  /api/audit                   ← paginated, query: orgId

POST /api/admin/requests/:id/approve   ← superadmin only
POST /api/admin/requests/:id/reject

POST /api/webhooks/stripe
```

---

## Next.js Pages

```
/                         ← marketing
/login                    ← Lucia auth
/signup
/request-org

/dashboard                ← all orgs, storage health cards
/org/[id]/overview        ← storage breakdown + active jobs + savings to date
/org/[id]/archive         ← filter form + preview + start job
/org/[id]/jobs            ← job history table
/org/[id]/jobs/[jobId]    ← live progress (polls /api/jobs/:id every 5s)
/org/[id]/restore         ← search D1 + view modal + restore button
/org/[id]/schedule        ← auto-archive rules
/org/[id]/audit           ← audit log + CSV export
/org/[id]/settings        ← org config, connection status

/admin/requests           ← pending org requests (superadmin)
/admin/orgs               ← all tenants (superadmin)

/sf-view/[caseId]         ← standalone page for SF custom button
                             validates SF session token from query param
                             shows archived email thread + attachments
```

---

## Edge Cases — Always Handle

### Validation Rules (SF)
allOrNone: false on ALL Composite and Bulk ingest calls.
FIELD_CUSTOM_VALIDATION_EXCEPTION → log to D1 sfFieldUpdateError, continue.
Data is still archived. Job does not abort.

### Flow Side Effects on Restore
Snapshot Case.Status before restore. Check after. Revert if changed.
Automatic, no client setup needed.

### Worker Memory — Attachment Streaming
```typescript
// CORRECT (zero memory):
const sfResp = await fetch(sfUrl, { headers })
await env.STORAGE.put(r2key, sfResp.body)

// WRONG (OOM on large files):
const bytes = await sfResp.arrayBuffer()
```

### Concurrency vs SF Limits
Production: max_concurrency 10, sandbox: max_concurrency 3.
Read from D1 sfOrgs.sfType to determine which.

### Idempotency
INSERT OR IGNORE on all jobRecords. UNIQUE(jobId, sfRecordId).
Safe to retry any failed Queue message at any time.

### CSV Streaming (phase1 CSVs can be 300MB–1GB)
Always stream-parse row by row. Never r2obj.text() for large files.

### JWT Bearer Errors
invalid_grant → username wrong or user not pre-authorized
app_not_found → Connected App not installed
inactive_user → integration user disabled
Show specific message per error code, never generic "auth failed".

---

## Build Status

```
Infrastructure:
[ ] monorepo scaffold
[ ] packages/db schema + migrations
[ ] packages/sf-client (jwt, bulk, composite, metadata, rest)
[ ] packages/types

Worker:
[ ] Hono app + middleware
[ ] Auth routes (Lucia)
[ ] SF OAuth + JWT setup routes
[ ] Jobs routes
[ ] View route (HTML render from R2)
[ ] CoordinatorWorkflow (all 3 phases)
[ ] BatchWorker (Queue consumer)
[ ] RestoreWorkflow
[ ] Cron handler

Web:
[ ] Marketing page
[ ] Auth pages
[ ] Org request flow
[ ] Dashboard
[ ] Archive page
[ ] Jobs + live progress
[ ] Restore page
[ ] Schedule page
[ ] Audit log
[ ] SF view page (/sf-view/[caseId])
[ ] Admin pages

Integrations:
[ ] Stripe subscriptions + metered usage
[ ] Stripe webhook
[ ] Email notifications
```

---

## Hard Rules — Never Violate

1. Every API route validates requesting user owns the orgId. No cross-tenant access.
2. SF_PRIVATE_KEY never leaves Worker environment. Never logged, never in responses.
3. Attachment binaries: always ReadableStream pipe. Never arrayBuffer() in Worker.
4. Queue max_concurrency: 10 production, 3 sandbox.
5. allOrNone: false on every Composite and Bulk ingest call.
6. INSERT OR IGNORE on all jobRecords inserts.