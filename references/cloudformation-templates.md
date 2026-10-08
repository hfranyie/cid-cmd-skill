# CloudFormation templates, versions, URLs, and regions

Template bucket, pinned versions, exact template URLs, per-stack parameter sets, and region
constraints. Extracted verbatim from `python/cid_mcp_external/catalog.py` and
`python/cid_mcp_external/cid_mcp_server.py`. The Python source is authoritative.

All stacks are public and unauthenticated — they resolve over the open internet with the
user's own AWS credentials. Every stack creates named IAM roles, so every deploy uses
`--capabilities CAPABILITY_NAMED_IAM` (no macros, so no `CAPABILITY_AUTO_EXPAND`).

## Live-validated behaviors & gotchas

Verified by actually deploying against a real account:

- **CloudFormation accepts the pinned template URL + `CAPABILITY_NAMED_IAM`** — the Data
  Exports stack create was accepted and began provisioning (mechanics confirmed).
- **One data-exports stack per account.** The Data Exports template publishes a **named
  CloudFormation Export** (`cid-DataExports-Database`). An account can hold only one stack
  exporting that name — a second `deploy_data_exports` in an account that already has one
  (e.g. an existing `CID-DataExports-Destination`) **fails** with:
  `Export with name cid-DataExports-Database is already exported by stack <name>`. This is
  CloudFormation enforcing export-name uniqueness, not a template bug. **Before deploying a
  Data Exports stack, check for an existing one** (`aws cloudformation list-stacks` and look
  for a `*DataExports*` stack, or `aws cloudformation list-exports` for the name). To run a
  second export deployment deliberately (side-by-side), use the **`ResourcePrefix`** parameter
  to namespace the exports. Verified live on an account that already had a Data Exports stack.
- **Deploy from a clean account for a true from-zero test.** The above means
  `deploy_data_exports` can only be validated from zero in an account with no prior CID
  exports footprint.
- **`--on-failure DELETE` rolls back cleanly.** When the create failed on the duplicate
  export, the stack auto-deleted to `DELETE_COMPLETE` leaving no resources behind — a safe
  default for test deploys. (Alternatively `--on-failure ROLLBACK` keeps the failed stack for
  inspection.)
- **Query a rolled-back/deleted stack by its ARN**, not its name — once a stack is
  `DELETE_COMPLETE`, `describe-stacks --stack-name <name>` returns "does not exist"; use the
  full stack ARN (returned by `create-stack`) to read its final status and events.
- **Multi-account data flow is S3 replication, and the destination must authorize the source
  FIRST (verified live).** The source account's export writes to the source's own local S3
  bucket, which **replicates** into the destination bucket — the destination bucket policy
  grants the source `s3:ReplicateObject`/`s3:ReplicateDelete`/`ListBucket`/`PutBucketVersioning`
  (policy Sid `AllowReplicationWrite`/`AllowReplicationRead`), NOT direct writes. So:
  1. The destination stack's `SourceAccountIds` must include the source account id. Updating
     that parameter (keep all others via `UsePreviousValue=true`) and letting the stack reach
     `UPDATE_COMPLETE` **does** rewrite the destination bucket policy to add the source
     (confirmed by reading the bucket policy after the update). Do this BEFORE deploying the
     source stack — otherwise the source export's bucket-permission validation fails with
     `AWS::BCMDataExports::Export ... S3 bucket permission validation failed`.
  2. Then deploy the source stack in the source account (same `ResourcePrefix`).
- **The source export writes DIRECTLY to the destination bucket (v0.12.0 model), and the
  destination bucket policy must grant the Data Exports service `s3:PutObject` — ROOT-CAUSED
  LIVE.** The v0.12.0 export resource (`LocalCUR2viaCFN`) has
  `S3Destination.S3Bucket = <prefix>-<destAcct>-data-exports` with `S3BucketOwner = <destAcct>`,
  and the Data Exports service validates write permission to that destination bucket at
  create time. The destination's v0.12.0 bucket policy provides this via the statement
  `Sid: EnableAWSDataExportsToWriteToS3` → `Principal: Service: bcm-data-exports.amazonaws.com`,
  `Action: s3:PutObject`, conditioned on `aws:SourceAccount ∈ SourceAccountIds`.
- **Version-mismatch failure (the real cause of the `S3 bucket permission validation failed`
  rollback — verified, and it is NOT a race condition).** If the destination account is running
  an **older** data-exports stack (observed: `v0.9.0`), its bucket policy only grants S3
  **replication** actions (`s3:ReplicateObject`/`ReplicateDelete`/`ListBucket`/
  `PutBucketVersioning`) and has **no** `EnableAWSDataExportsToWriteToS3` / `s3:PutObject`
  statement and no `S3BucketOwner` direct-delivery support. A **v0.12.0 source** export then
  fails validation against that old destination bucket. **Updating `SourceAccountIds` alone
  does not fix it** — with `--use-previous-template` the stack keeps the old template, so the
  PutObject grant is never added. **The fix: update the destination stack to the v0.12.0
  template** (`--template-url <...0.12.0...data-exports-aggregation.yaml>`, NOT
  `--use-previous-template`). Verified live: a v0.9.0 → v0.12.0 destination `update-stack`
  reached `UPDATE_COMPLETE` in place, the bucket policy gained the
  `EnableAWSDataExportsToWriteToS3` `s3:PutObject` grant, and the previously-failing source
  export stack then reached `CREATE_COMPLETE`. **Before deploying a source against an existing
  destination, confirm the destination is on a matching (current) template version** — check
  its deployed template `Description` (`aws cloudformation get-template ... | grep Description`
  shows e.g. `... Aggregation v0.9.0`).
- **Correct ordering for a multi-account source→destination export (verified end-to-end):**
  1. Destination account: ensure the destination stack is on the **v0.12.0 template** and its
     `SourceAccountIds` includes the source account id (update with the template URL if it is
     older). Reach `UPDATE_COMPLETE`; confirm the bucket policy has the `s3:PutObject` grant.
  2. Source account: deploy the v0.12.0 template with `DestinationAccountId=<dest>`,
     `SourceAccountIds=<source>`, matching `ResourcePrefix`, `ManageCUR2=yes`. The export
     (`LocalCUR2viaCFN`) + local `SourceS3` reach `CREATE_COMPLETE`.

## Template bucket and pinned versions

- Bucket: `aws-managed-cost-intelligence-dashboards`
- Base URL: `https://aws-managed-cost-intelligence-dashboards.s3.amazonaws.com`

Pinned versions (bump deliberately; floating on "latest" is intentionally avoided):

| Family | Version |
|---|---|
| data-collection | `v3.14.8` |
| data-exports | `0.12.0` |
| cur-aggregation (legacy CUR 1.0) | `0.2.0` |
| cid-cfn.yml / cid-cmd | `4.4.17` |

## Exact template URLs

Built from the version constants above:

- Data Collection read-permissions:
  `https://aws-managed-cost-intelligence-dashboards.s3.amazonaws.com/cfn/data-collection/v3.14.8/deploy-data-read-permissions.yaml`
- Data Collection stack:
  `https://aws-managed-cost-intelligence-dashboards.s3.amazonaws.com/cfn/data-collection/v3.14.8/deploy-data-collection.yaml`
- Data Collection management-account role (referenced by catalog):
  `https://aws-managed-cost-intelligence-dashboards.s3.amazonaws.com/cfn/data-collection/v3.14.8/deploy-in-management-account.yaml`
- Data Collection linked-account role (referenced by catalog):
  `https://aws-managed-cost-intelligence-dashboards.s3.amazonaws.com/cfn/data-collection/v3.14.8/deploy-in-linked-account.yaml`
- Data Exports aggregation (CUR 2.0 / FOCUS / Cost Optimization Hub):
  `https://aws-managed-cost-intelligence-dashboards.s3.amazonaws.com/cfn/data-exports/0.12.0/data-exports-aggregation.yaml`
- Legacy CUR 1.0 aggregation:
  `https://aws-managed-cost-intelligence-dashboards.s3.amazonaws.com/cfn/data-exports/0.2.0/cur-aggregation.yaml`
- CFN-native dashboard deploy:
  `https://aws-managed-cost-intelligence-dashboards.s3.amazonaws.com/cfn/4.4.17/cid-cfn.yml`

## Per-stack parameters and default stack names

Deploy with the AWS CLI, e.g.:

```bash
aws cloudformation deploy \
  --template-url <template-url> \
  --stack-name <stack-name> \
  --parameter-overrides Key1=Value1 Key2=Value2 \
  --capabilities CAPABILITY_NAMED_IAM \
  [--profile <profile>] [--region <region>]
```

### Data Exports — destination vs. source accounts (verified against the template)

**The same `data-exports-aggregation.yaml` template is deployed in BOTH the destination
account and each source account, tied together by a matching `ResourcePrefix`.** The template
figures out which role it is playing from its own conditions (read verbatim from the 0.12.0
template):

- `IsDestinationAccount` = true when `DestinationAccountId == AWS::AccountId`. The destination
  is where exports land and where you run dashboards / Data Collection. Its stack creates the
  landing S3 bucket + Glue/Athena.
- `IsSourceAccount` = true when the stack runs in an account that is **not** the destination,
  **or** it is the destination **and** that account id is listed **first** in
  `SourceAccountIds`. The source side is what actually **creates the CUR/FOCUS/COH export and
  replicates it** to the destination.
- **`ResourcePrefix` MUST be identical in the destination and source stacks** (template's own
  words: "Must be the same in destination and source stacks") — it is the link between them.

Two real topologies:

- **Single account (Source = Destination):** deploy ONE stack with `DestinationAccountId` and
  `SourceAccountIds` set to the **same** account id (the template says: put the destination
  account id **first** in `SourceAccountIds`). That one stack both creates the export and sets
  up the landing. This is the clean from-zero case for a standalone test account.
- **Multi-account:** deploy the template in the **destination** once (sets up landing), and
  deploy it again in **each source account** (`DestinationAccountId=<dest>`,
  `SourceAccountIds=<that source>` or the full list, same `ResourcePrefix`) to create and
  replicate that account's export into the destination. An account that already holds the
  destination stack (e.g. an existing `CID-DataExports-Destination`) must NOT get a second
  destination stack — see the one-stack-per-account export conflict in "Live-validated
  behaviors & gotchas" above; add source accounts by deploying in THOSE accounts instead.

- Default stack name: `cid-data-exports`
- Template: `data-exports-aggregation.yaml`
- Parameters:
  - `DestinationAccountId` — 12-digit account where exports land (required)
  - `ManageCUR2` = `yes` | `no` (default `yes`; needed for Foundational dashboards)
  - `ManageFOCUS` = `yes` | `no` (default `no`; needed for the FOCUS dashboard)
  - `ManageCOH` = `yes` | `no` (default `no`; Cost Optimization Hub, needed for CORA;
    requires COH already enabled)
  - `SourceAccountIds` — optional, comma-separated 12-digit account IDs whose exports
    replicate to the destination

#### Data Exports — full parameter surface (advanced)

The five parameters above are the **common path**. The template's `Parameters:` block exposes
more; names and allowed values below are verbatim from `data-exports-aggregation.yaml`. All
are mutating — the confirm-first framing applies. (Note: `DataExports`, `IsSourceAccount`,
`DeployDataExport`, `DeployWriteBlocker` are template **Conditions/Mappings**, not parameters,
so you cannot pass them — only the parameters below exist.)

**(a) Data-shape toggles — change what the export contains:**

- `ManageCarbon` = `yes` | `no` (default `no`) — add a carbon-emissions export; feeds the
  Sustainability dashboard.
- `EnableSCAD` = `yes` | `no` (default `yes`) — Split Cost Allocation Data (container/ECS cost
  allocation). Set `no` if dataset size causes performance issues.
- `EnableIAMPrincipalData` = `yes` | `no` (default `yes`) — include IAM principal (caller
  identity) attribution for Bedrock inference cost; adds a `line_item_iam_principal` column.
- `CUR2TimeGranularity` = `HOURLY` | `DAILY` | `MONTHLY` (default `HOURLY`) — CUR 2.0
  granularity. **Changing it requires redeploy + purge of destination data + a backfill
  request**; `HOURLY` recommended unless the invoice exceeds ~$50M.
- `FOCUSTimeGranularity` = `HOURLY` | `DAILY` | `MONTHLY` (default `HOURLY`) — same for the
  FOCUS export, same redeploy/purge/backfill caveat.

**(b) Deployment-shape — where/how it lands:**

- `ResourcePrefix` (default `cid`) — prefix for all named resources incl. the S3 bucket.
  **MUST be the same in the destination and source stacks** (max 37 chars, lowercase/digits/
  hyphens).
- `RolePath` (default `/`) — IAM role path; needed under permission-boundary / SCP regimes.
- `LakeFormationEnabled` = `yes` | `no` (default `no`) — register the data in Lake Formation;
  requires `cid-lakeformation-prerequisite.yaml` installed first. Leave `no` if unsure.
- `LegacyLocalBucket` = `yes` | `no` (default `yes`) — keep `yes` when updating from a previous
  version (retains the local S3 bucket); set `no` for brand-new deployments.
- `SecondaryDestinationBucket` (default empty) — optional bucket name to replicate export data
  to a second destination. Keep empty if unsure.

**(c) Write-blocker scheduling (advanced / rarely needed) — pause export writes during
QuickSight refresh:**

- `AddScheduleForBlockingWrite` = `yes` | `no` (default `no`) — schedule a window that stops
  Data Export writes so a QuickSight dataset refresh is consistent.
- `DisableWriteCronSchedule` (default `0 1 * * ? *`, UTC cron) — when to disable writes.
- `EnableWriteCronSchedule` (default `0 3 * * ? *`, UTC cron) — when to re-enable writes.

### Legacy CUR 1.0 aggregation — prefer Data Exports for new setups

- Default stack name: `cid-cur-aggregation`
- Template: `cur-aggregation.yaml`
- Parameters:
  - `DestinationAccountId` — 12-digit account where the CUR lands (required)
  - `SourceAccountIds` — optional, comma-separated 12-digit account IDs

### Data Collection permissions (Step 1) — run in the Organizations MANAGEMENT account

- Default stack name: `cid-data-collection-permissions`
- Template: `deploy-data-read-permissions.yaml`
- Deploy timeout: 30 minutes (StackSet fan-out across an org can be slow)
- Parameters:
  - `DataCollectionAccountID` — 12-digit account ID where the Data Collection stack runs
    (NOT the management account)
  - `OrganizationalUnitIds` — comma-separated OU ID(s) to fan the read role out to via
    StackSets (use the Organization Root ID for full visibility). Requires AWS
    Organizations trusted access + service-managed StackSets enabled.
  - one `IncludeXModule=yes` per module that needs a cross-account role. Reject modules that
    have no cross-account parameter (`org_data`, `quicksight`, `reference`, `aws_feeds`,
    `isv_feeds`, `kiro_user_activity`) — see
    [data-collection-modules.md](./data-collection-modules.md).
- Advanced (non-module) parameters, from `deploy-data-read-permissions.yaml`:
  - `MultiAccountRoleName` (default `Optimization-Data-Multi-Account-Role`) — the read-only
    role fanned out to linked accounts. **MUST match the collection stack** (see coupling note
    below).
  - `ManagementAccountRole` (default `Lambda-Assume-Role-Management-Account`) — role assumed in
    the management account. **MUST match the collection stack.**
  - `ResourcePrefix` (default `CID-DC-`) — prefix on all created roles. **MUST match the
    collection stack.**
  - `AllowModuleReadInMgmt` — allow creating module read roles in the **management account
    itself**, so you can collect management-account data too, not just linked accounts.
  - `CFNSourceBucket` (default `aws-managed-cost-intelligence-dashboards`) — template/Lambda
    source bucket. **DO NOT CHANGE** except in advanced/air-gapped mirrors.

### Data Collection (Step 2) — run in a dedicated DATA COLLECTION account

- Default stack name: `cid-data-collection`
- Template: `deploy-data-collection.yaml`
- Deploy timeout: 25 minutes
- Must be deployed in a supported region (see list below)
- Parameters:
  - `ManagementAccountID` — comma-separated 12-digit Payer/Management account ID(s) to
    collect data from (required)
  - `RegionsInScope` — optional, comma-separated AWS regions to collect from (empty =
    current region only)
  - one `IncludeXModule=yes` per enabled module
  - Advanced (non-module) parameters, from `deploy-data-collection.yaml`:
    - `Schedule` (default `rate(14 days)`) — EventBridge cadence for the less-frequent modules
      (Trusted Advisor, Compute Optimizer, Organizations data, Rightsizing, RDS Utilization,
      Inventory, Transit Gateway, Backup, ECS Chargeback). Do not go above once per day; more
      frequent collection costs more.
    - `ScheduleFrequent` (default `rate(1 day)`) — cadence for the frequent modules (Cost
      Anomalies, Budgets, Support Cases, Health Events). Same cost caveat.
    - `DatabaseName` (default `optimization_data`) — target Athena/Glue database for collected
      data.
    - `ResourcePrefix` (default `CID-DC-`) — prefix on all created resources. **CANNOT be
      updated in place** (delete and re-create the stack to change it) and **MUST match the
      permissions stack**.
    - `MultiAccountRoleName` (default `Optimization-Data-Multi-Account-Role`) — the
      cross-account read role name. **MUST match the permissions stack.**
    - `ManagementAccountRole` (default `Lambda-Assume-Role-Management-Account`) — role assumed
      in the management account. **MUST match the permissions stack.**
    - `DataBucketsKmsKeysArns` (default empty) — comma-separated KMS key ARNs for encrypted
      data buckets / Glue Catalog; `*` grants decrypt on all keys. Leave empty if nothing is
      KMS-encrypted.
    - `CFNSourceBucket` (default `aws-managed-cost-intelligence-dashboards`) — template/Lambda
      source bucket. **DO NOT CHANGE** except in advanced/air-gapped mirrors.

> **Cross-stack coupling (easy to get wrong, fails silently).** `MultiAccountRoleName`,
> `ManagementAccountRole`, and `ResourcePrefix` **MUST be identical** between the
> read-permissions stack (Step 1) and the collection stack (Step 2). If they drift apart, the
> collection account tries to assume a role name the permissions stack never created, and
> cross-account reads fail with no obvious error. If you override any of these on one stack,
> override it to the same value on the other.

### CFN dashboard deploy — alternative to `cid-cmd deploy`

- Default stack name: `Cloud-Intelligence-Dashboards`
- Template: `cid-cfn.yml`
- Deploy timeout: 25 minutes
- Covers only 5 dashboards: CUDOS v5, Cost Intelligence, KPI, TAO, Compute Optimizer (not
  the full catalog)
- Requires QuickSight Enterprise Edition with SPICE capacity already activated in the region
- Parameters:
  - `PrerequisitesQuickSight` = `yes`
  - `PrerequisitesQuickSightPermissions` = `yes`
  - `QuickSightUser` — QuickSight username (as shown in the QuickSight admin panel) that
    will own the dashboards (required)
  - `CURVersion` — `2.0` (default) or `1.0`
  - `DeployCUDOSv5` = `yes` | `no`
  - `DeployCostIntelligenceDashboard` = `yes` | `no`
  - `DeployKPIDashboard` = `yes` | `no`
  - `DeployTAODashboard` = `yes` | `no`
  - `DeployComputeOptimizerDashboard` = `yes` | `no`
  - `ShareDashboard` = `yes` | `no`
  - `OptimizationDataCollectionBucketPath` — `s3://` path from your Data Collection stack;
    required when `DeployTAODashboard=yes` or `DeployComputeOptimizerDashboard=yes`

## Regions

### Data Collection supported regions (18)

The Data Collection CFN stacks can only be deployed in these regions (from
`deploy-data-collection.yaml`'s own `Mappings.RegionMap`). This is stricter than the general
"QuickSight is available" list; deploying elsewhere is refused with a clear message:

```
ap-northeast-1  ap-northeast-2  ap-south-1   ap-southeast-1  ap-southeast-2
ca-central-1    eu-central-1    eu-central-2 eu-north-1      eu-south-1
eu-west-1       eu-west-2       eu-west-3    sa-east-1
us-east-1       us-east-2       us-west-1    us-west-2
```

### CFN-native CUR regions (2)

Data Exports / legacy CUR can be created natively via CloudFormation resources only in
`us-east-1` and `cn-northwest-1`. Everywhere else the `data-exports-aggregation.yaml` /
`cur-aggregation.yaml` templates transparently fall back to a Lambda-backed custom resource
— deployment still works, it just takes a different (slower) internal code path. This is
worth surfacing to a user who asks "why did this take longer in region X".
