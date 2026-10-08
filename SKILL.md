---
name: cid-cmd-skill
description: >-
  Deploy and manage AWS Cloud Intelligence Dashboards (CUDOS, Cost Intelligence, KPI,
  Trusted Advisor/TAO, CORA, Graviton, Health Events, FOCUS, and the rest of the CID
  catalog) in the user's own AWS account by driving the public cid-cmd CLI and public
  CloudFormation templates with the user's own AWS credentials. Use when a user wants to
  deploy, update, delete, migrate, check the status of, or share CID/QuickSight cost
  dashboards, or set up the CUR 2.0 data exports and advanced Data Collection pipeline that
  feed them.
version: 1.1.0
tags:
  - aws
  - cid
  - cudos
  - quicksight
  - cost
  - dashboards
  - cid-cmd
  - cloudformation
  - skill
---

# CID dashboards

AWS Cloud Intelligence Dashboards (CID) are Amazon QuickSight dashboards — CUDOS, Cost
Intelligence, KPI, Trusted Advisor (TAO), CORA, Graviton, Health Events, FOCUS, and more —
that turn Cost and Usage data into cost-optimization insight. This skill tells you how to
deploy and manage them yourself by running two public tools directly:

- the **`cid-cmd`** CLI (`pip install cid-cmd`) for dashboard lifecycle, and
- **CloudFormation** (public templates) for the underlying infrastructure: CUR 2.0 / FOCUS /
  Cost Optimization Hub data exports and the advanced Data Collection pipeline.

There is no MCP server and no special account. You act with the user's own AWS credentials.
This skill is the conversion of a former MCP server: the server's tools are now documented
procedures you perform yourself with your shell / AWS tools.

## Prerequisites

- **Python 3.10+** and `pip install cid-cmd`.
- **Locate `cid-cmd` — do NOT assume it is on PATH.** `pip install cid-cmd` installs a
  console script, but it may land in a directory that is not on `PATH` — most notably on
  macOS **python.org framework builds**, where it goes to
  `/Library/Frameworks/Python.framework/Versions/<X.Y>/bin/cid-cmd`. Before running anything,
  LOCATE the executable (mirroring the former MCP server's `resolve_cid_cmd_path()`), in this
  order: (1) the active venv's `bin/` (`<venv>/bin/cid-cmd`); (2) `which cid-cmd` on PATH;
  (3) the interpreter's scripts dir — `python3 -c "import sysconfig; print(sysconfig.get_path('scripts'))"`
  — and the macOS framework bin path above. Use the full path you find rather than assuming
  bare `cid-cmd` resolves.
- The user's **own AWS credentials** (a profile, SSO session, or ambient env/instance role)
  with IAM permissions for QuickSight, Athena, Glue, S3, IAM, and CloudFormation.
- **QuickSight Enterprise Edition with SPICE capacity activated** in the target region
  **before** deploying any dashboard. cid-cmd will not provision QuickSight automatically
  during a deploy — but it CAN bootstrap QuickSight Enterprise for you via `cid-cmd init-qs`
  (mutating and billable — see
  [references/cid-cmd-subcommands.md](./references/cid-cmd-subcommands.md)).
- **Outbound HTTPS to `raw.githubusercontent.com`.** cid-cmd fetches its extended dashboard
  catalog (CORA, TAO, Graviton, Health Events, etc.) from GitHub on every run. If that is
  blocked, catalog dashboards fail with "unknown dashboard-id" even though cid-cmd works.

## Authentication

Exactly one model: the user's own AWS credentials via boto3's standard chain (environment,
`~/.aws` profiles, SSO, instance/task role). Every operation takes an **optional profile and
region**:

- `cid-cmd`: pass `--profile <profile>` and `--region_name <region>`.
- `aws` CLI: pass `--profile <profile>` and `--region <region>`.

When a profile is given, scrub `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`,
`AWS_SESSION_TOKEN`, and `AWS_SECURITY_TOKEN` from the environment first, so a stale exported
key cannot silently shadow the profile the user asked for. If no region is given, let the
AWS tools resolve it from the profile/environment — do not invent one, because guessing wrong
targets the wrong place for CloudFormation.

Verify credentials before anything else with `aws sts get-caller-identity`.

## Safety model (READ THIS BEFORE ANY MUTATING ACTION)

The original MCP server enforced a `confirm=true` gate in code. In this skill those gates are
**PROMPT-enforced, not code-enforced** — there is no backstop but your own discipline. Be
disciplined.

- You **MUST** obtain explicit user confirmation before any mutating action: deploy, update,
  delete, migrate, or share a dashboard; create, update, or delete any CloudFormation stack.
- You **MUST** clearly state that the action is **real and billable** in the user's AWS
  account before you run it.
- You **MUST** prefer read-only operations (status, check-updates, list dashboards, get stack
  status, list stacks, diagnostics) when the user is only inspecting. Those **MAY** run
  without confirmation.
- You **MUST** warn that `update --recursive` and `--on-drift override` can **overwrite
  dashboard and Athena-view customizations**.
- You **MUST** warn that `cid-cmd delete` and CloudFormation **stack deletion are destructive
  and irreversible**.
- You **MUST NOT** disable safety protections or delete production stacks without explicit
  user direction. When you cannot tell whether a resource is production, assume it is and ask.

## Judging success and failure of cid-cmd

**Exit code 0 does NOT mean success.** `cid-cmd`'s CLI wrapper (the `cid_command` decorator
in `cid/cli.py`) catches its own `CidError` / `CidCritical` exceptions, logs them, and
returns normally — so the process exits **0 even when the command failed**. Do not trust the
exit code alone.

The reliable failure signal is a **log line**, not the exit code. Its console log format is
fixed as `%(levelname)s - %(message)s`, so apply this rule (it mirrors `_FAILURE_LOG_RE` in
the former MCP server's `cid_cmd_runner.py`):

- Scan stdout. Treat the run as **FAILED** if **either**:
  - any line matches `^(ERROR|CRITICAL) - ` (i.e. begins with `ERROR - ` or `CRITICAL - `), **or**
  - the process exits **non-zero** (genuine crashes cid-cmd does not catch — e.g. a missing
    AWS profile — do exit 1).
- Otherwise treat it as **success**.

There is exactly **one** documented exception to "non-zero exit = failure": the `status`
picker traceback below, which exits non-zero but is NOT a real failure and does NOT contain
an `ERROR - ` / `CRITICAL - ` line. Reconcile the two rules in that order: the status-picker
recognition below wins for `status`; everywhere else, apply the failure-log rule.

### The `status` picker-traceback recognition rule (treat as SUCCESS)

> **ALWAYS run `status` with stdin closed and a timeout.** After printing the status,
> `cid-cmd status` opens an interactive action menu (Back / Open / Refresh / Update / Exit).
> How that menu behaves depends on the shell: with **no TTY** it raises immediately and the
> process exits (the traceback case below); but in a shell that gives it a **pseudo-TTY**
> (common for agent/IDE terminals) the menu *renders and blocks, waiting for a keypress*, and
> the command hangs until it is killed. To make the behavior deterministic either way, append
> `< /dev/null` (close stdin) and bound it with a timeout. Verified live: without `< /dev/null`
> the same command that cleanly tracebacks on one machine hangs for minutes on another.
>
> ```bash
> # read-only status, hardened against the interactive-picker hang:
> cid-cmd --region_name <r> -y status --dashboard-id <id> < /dev/null
> # (optionally wrap with a timeout, e.g. a 180s cap, since status still does real AWS calls)
> ```
>
> With stdin closed, the picker cannot block — it degrades to the clean traceback case, which
> you read with the rule below.

`cid-cmd status` and `cid-cmd status --dashboard-id <id>` print the **real** status and then
may emit a Python traceback. This is a known upstream defect, not a failure: `status()` in
`cid/common.py` calls `display_status()` + `display_url()` and THEN enters an interactive
action-menu loop — `get_parameter(...)` with choices **Back / Open / Refresh / Update /
Exit** — which, with no TTY (headless/non-interactive), raises. So the status lookup has
already **completed successfully** before the traceback is produced.

**Recognition rule the agent MUST apply.** Treat the run as **SUCCESS** when stdout contains
a complete `Dashboard status:` block followed by that specific trailing exception:

1. a line `Dashboard status:`, then the fields `Id: <id>`, `Health: ...`, `Status: ...`,
   `Version: ...` (a healthy block also shows `Datasets:` and the dashboard URL
   `https://.../sn/dashboards/<id>`), **followed by**
2. a Python traceback ending in exactly:
   `Exception: Please set parameter <id>. Unable to request user in environment={...}`
   (where `<id>` is the dashboard id you asked about).

When you see **a complete `Dashboard status:` block (with `Id:`, `Health:`, `Status:`,
`Version:`) followed by that trailing exception = SUCCESS.** Report the status block above the
traceback as the valid result and do **NOT** report failure. (This is the same logic the
former MCP server applied as `_completed_status_before_headless_picker` — its origin and name
of record.)

Do NOT apply this leniency to any other output: if the `Dashboard status:` block is
incomplete, or the trailer is a different exception, or an `ERROR - ` / `CRITICAL - ` line
appears in the status text itself, it is a real failure — report it.

**Critical: `Please set parameter <X>` is only benign when `<X>` is the dashboard id after a
complete status block.** `deploy`/`update` can fail with the *same phrasing* for a different
parameter — e.g. `Please set parameter athena-database`, `athena-workgroup`,
`athena-result-bucket`, `quicksight-datasource-id`, or `cur-table-name-and-db` — when run
unattended without that parameter. Those are **REAL failures**: cid-cmd needed a value, had
no TTY to prompt, and aborted *before completing the operation* (there is no completed result
block above them). Verified live. The fix is to supply the missing parameter (see
"Required parameters for unattended deploy/update" below), not to treat it as benign.

**Non-fatal `ERROR - ... Will continue without` warnings.** Also verified live: `deploy` can
emit `ERROR - ` lines that cid-cmd *recovers from*, each ending `... Will continue without.`
(e.g. a stale CUR view blocking optional tag-scan queries). These are warnings, not the
operation failing. So the raw `^(ERROR|CRITICAL) - ` scan can produce a **false FAILED** here.
Resolve it by also checking for a positive completion signal: a successful deploy/update ends
with `Update completed` or `Dashboard <id> deleted` / `... is available at: https://.../sn/dashboards/<id>`,
**and** the dashboard then actually appears in `aws quicksight list-dashboards`. When you see
`ERROR - ` lines but every one is immediately followed by `Will continue without` AND the
completion signal is present AND the dashboard is listed, treat it as success-with-warnings;
otherwise treat `ERROR -`/`CRITICAL -` as failure. When in doubt, confirm with a follow-up
`status --dashboard-id <id>` or `list-dashboards` — the deployed state is the source of truth.

Run cid-cmd from a scratch temp directory: it writes a `cid.log` into the current working
directory, and you do not want that in the user's project.

## Choosing a dashboard

Resolve the user's phrase to a canonical `--dashboard-id` using
[references/dashboard-catalog.md](./references/dashboard-catalog.md). That file has the full
id list, the friendly-name aliases, and which dashboards need the Data Collection pipeline.
Note `cudos` (friendly) → `cudos-v5`, while the exact id `cudos` is the deprecated v4.

## Read-only operations

All read-only; no confirmation required.

- **Overall status / credential check:** `cid-cmd --profile <p> --region_name <r> -y status < /dev/null`
- **Check a dashboard for updates:**
  `cid-cmd --profile <p> --region_name <r> -y status --dashboard-id <id> < /dev/null`
  (the `< /dev/null` is REQUIRED — see the status picker-hang note under "Judging success and
  failure"; without it the command can hang on an interactive menu in a TTY shell).
- **List deployed dashboards:** cid-cmd has no non-interactive list subcommand, so use
  QuickSight directly:
  `aws quicksight list-dashboards --aws-account-id <acct> [--profile <p> --region <r>]`
  (lists ALL QuickSight dashboards, not just CID's — check the region if you expect CID ones).
- **CFN stack status:** `aws cloudformation describe-stacks --stack-name <name> ...`
- **List CFN stacks:** `aws cloudformation list-stacks ...` (filter by a `cid` name prefix to
  surface stacks created here).
- **Diagnostics:** confirm `cid-cmd` is installed (`cid-cmd --help` → first line shows the
  version, e.g. `... CLI 4.4.17`); confirm credentials resolve
  (`aws sts get-caller-identity`); note the pinned template versions (see
  [references/cloudformation-templates.md](./references/cloudformation-templates.md)); and
  confirm outbound HTTPS to `raw.githubusercontent.com` is reachable.

## Deploy a dashboard (`cid-cmd deploy`)

Mutating — confirm first. Pattern:

```bash
cid-cmd --profile <p> --region_name <r> -y deploy --dashboard-id <id> [options]
```

Allowlisted options the former MCP server exposed, with allowed values/constraints:

- `--athena-database <name>`
- `--athena-workgroup <name>`
- `--glue-data-catalog <name>`
- `--cur-table-name <name>` (the actual CUR Athena table name; not a CUR version)
- `--quicksight-datasource-id <id-or-arn>`
- `--quicksight-datasource-role-arn <arn>`
- `--allow-buckets <bucket1,bucket2,...>` (comma-separated S3 buckets granted to the default
  CID QuickSight role)
- `--quicksight-user <user>`
- `--quicksight-group <group>`
- `--catalog <files-or-urls>` (comma-separated cid-cmd catalog files or URLs)
- `--account-map-source <csv|dummy|organization>` (lowercased). With `csv`,
  `--account-map-file <existing.csv>` is **required**; `--account-map-file` is valid only when
  the source is `csv`.
- `--account-map-file <existing csv path>` (must exist locally)
- `--resources <local-yaml | https:// URL>` (a local path must exist; otherwise an `https://`
  URL). Used to deploy custom dashboards like `unit-cost-dashboard` and `shard`.
- `--on-drift <show|override>` (lowercased; `override` **replaces** customizations)
- `--theme <CLASSIC|MIDNIGHT|SEASIDE|RAINIER>`
- `--currency <USD|GBP|EUR|JPY|KRW|DKK|TWD|INR>`
- `--update <yes|no>` (whether cid-cmd updates existing dashboard elements)
- Dataset overrides: emit one `--dataset-<name>-id <value>` per override (names limited to
  letters, digits, underscore, hyphen).
- View parameter overrides: emit one `--view-<view>-<param> <value>` per override (view and
  param names limited to letters, digits, underscore, hyphen).
- `--rls <CLEAR|ENABLED|DISABLED>` with `--rls-dataset-id <id>` **required iff** `ENABLED`
  (and only valid when `ENABLED`).
- `--share-with-account` (boolean flag)
- `--quicksight-delete-failed-datasource` (boolean flag)

### Required parameters for unattended deploy/update (verified live)

The options above are listed by the upstream CLI as optional, but running headless (no TTY)
`deploy`/`update` **aborts asking for them one at a time** if they are missing — each as
`Please set parameter <X>. Unable to request user ...` (a REAL failure, see "Judging success
and failure"). You must supply the whole chain up front. For a CUR-based dashboard (CUDOS,
Cost Intelligence, KPI) the working set is:

```bash
cid-cmd --region_name <r> -y deploy --dashboard-id <id> \
  --athena-database <db> \
  --athena-workgroup <wg-with-a-result-location> \
  --quicksight-datasource-id <qs-athena-datasource-id> \
  --cur-table-name <cur-table-or-view>
```

The parameters cid-cmd prompts for, in the order they surface, and how to discover each:

- `--athena-database` — list with `aws glue get-databases --query "DatabaseList[].Name"`.
  For CUR dashboards it is the CUR database (commonly `cid_cur`).
- `--athena-workgroup` — list with `aws athena list-work-groups`. **Pick one that already has
  a result OutputLocation configured** (check `aws athena get-work-group --work-group <wg>
  --query "WorkGroup.Configuration.ResultConfiguration.OutputLocation"`). A workgroup with no
  result location (often `primary`) makes cid-cmd try to create one and then prompt for
  `athena-result-bucket`. CID deployments usually have a purpose-built `CID` workgroup — use it.
- `--quicksight-datasource-id` — needed on a **fresh deploy** that must create datasets
  (an `update` of an existing dashboard whose datasets exist does not re-prompt for it). List
  with `aws quicksight list-data-sources --aws-account-id <acct> --query
  "DataSources[?Type=='ATHENA'].[DataSourceId,Name]"` (commonly `CID-Athena`).
- `--cur-table-name` — the CUR Athena table/view the dashboard queries. List tables with
  `aws glue get-tables --database-name <db> --query "TableList[].Name"`; for CUR 2.0 it is
  typically the `cur2_view` view (not the raw `cur2` table).
- `--timezone <IANA tz>` — only needed when the operation migrates a dataset to the new
  QuickSight Data Preparation Experience (recursive updates, and some deploys); supply e.g.
  `America/New_York` to avoid the `Please set parameter timezone` abort.

Verified live end-to-end: with the full chain above (plus a healthy CUR view), a from-scratch
`deploy --dashboard-id cudos-v5` creates the datasets and lands a healthy dashboard
(`Health: healthy, Status: up to date`).

An **`update` of an already-deployed dashboard** typically only needs `--athena-database` and
`--athena-workgroup` (its datasource/datasets/CUR table are already recorded), but supply the
full chain if an update prompts for more. **`delete` needs none of these** — `delete
--dashboard-id <id>` works unattended on its own.

Deploy-time guidance to pass on to the user:

- Advanced dashboards (see the catalog's "Needs Data Collection" column) show no data until
  the Data Collection pipeline has run — set that up first (see Advanced workflow below).
- `cora` needs AWS Cost Optimization Hub enabled **and** a COH data export.
- `focus-dashboard` needs a FOCUS data export.
- `VIEW_IS_STALE ... stored view column count (N) does not match` on the CUR view means the
  CUR Athena view is stale: schema drift between the view and its underlying table (the CUR
  export gained columns but the view's stored schema was not regenerated — verified live, a
  view frozen at 73 columns over a 130-column table). During a fresh `deploy` this is a HARD
  blocker — creating the required datasets reads the view and fails — even though the earlier
  optional tag-scan queries only logged `ERROR - ... Will continue without.`. **Fix: recreate
  the view so its stored schema re-derives from the current table.** A CID CUR view is simply
  `SELECT * FROM "<cur-export-db>"."<cur-table>"` (e.g. `SELECT * FROM
  "cid_data_export"."cur2"`); read its exact definition from
  `aws glue get-table --database-name <db> --name <view> --query Table.ViewOriginalText`
  (Presto views base64-encode the SQL after the `Presto View:` marker), then run
  `CREATE OR REPLACE VIEW <view> AS <that same select>` in Athena against a workgroup with a
  result location. Re-probe with `SELECT COUNT(*) FROM <db>.<view>` and confirm the Glue
  stored column count now matches. Then redeploy. (`update --recursive` also rebuilds
  datasets/Athena views — but `--recursive` is a flag of **`update`, NOT `deploy`**; passing
  it to `deploy` is rejected with `Unknown extra argument`. Recursive rebuilds are slow (can
  exceed 15 min); run them in the background.)
- A recursive `update` (and any deploy/update that migrates a dataset to the **new QuickSight
  Data Preparation Experience**) additionally prompts for **`--timezone <IANA tz>`** (e.g.
  `America/New_York`) for the dataset refresh schedule, and aborts unattended without it
  (`Please set parameter timezone...`). Include `--timezone` in the chain for those paths.
  Note the migration message "will delete and recreate this dataset, temporarily breaking the
  dashboards listed above" — recursive updates briefly break dashboards sharing those datasets
  until each is re-updated. Verified live.

## Update a dashboard (`cid-cmd update`)

Mutating — confirm first.

```bash
cid-cmd --profile <p> --region_name <r> -y update [options]
```

- `--force` (allow updating an already up-to-date dashboard)
- `--recursive` (update datasets and Athena views — **can overwrite customizations**)
- (`force_recursive` in the old tool was a deprecated shortcut that set **both** `--force`
  and `--recursive`.)
- `--dashboard-id <id>` (optional; omit only when unattended selection is acceptable)
- `--on-drift <show|override>`, `--theme <...>`, `--currency <...>`, and `--rls` /
  `--rls-dataset-id` exactly as in deploy.
- **Unattended `update` also needs the Athena parameter chain** (`--athena-database`,
  `--athena-workgroup` with a result location; and the datasource/CUR params if it prompts).
  See "Required parameters for unattended deploy/update" above — the same prompt-then-abort
  behavior applies.

## Migrate to CUR 2.0

Mutating — confirm first. This is exactly:

```bash
cid-cmd --profile <p> --region_name <r> -y update --force --recursive
```

It repoints existing dashboards at the CUR 2.0 table. It **WILL override Athena view
customizations**. A CUR 2.0 Data Export must already exist and be delivering data (it does
not create the export). Set up the export first via the Data Exports stack.

## Delete a dashboard (`cid-cmd delete`)

Mutating and **destructive/irreversible** — confirm first.

```bash
cid-cmd --profile <p> --region_name <r> -y delete --dashboard-id <id> [--athena-database <name>]
```

Deletes the dashboard and its unshared dependencies. (Dashboards deployed via CloudFormation
are not cid-cmd dashboards — delete those by deleting the stack instead.)

## Sharing is currently broken on cid-cmd 4.4.17

Automated sharing does **not** work. In cid-cmd 4.4.17 the `share` subcommand parses
`--share-method`, `--folder-method`, `--folder-id`, `--folder-name`, `--quicksight-user`, and
`--quicksight-group`, but its body ignores them (it calls the share routine with only the
dashboard id and silently drops the rest — unlike deploy/update/delete, which forward their
options). Run non-interactively it then fails on a prompt it cannot satisfy
(`Exception: Please set parameter share-method. Unable to request user ...`).

- You **MUST NOT** claim a share succeeded via automation.
- Instead, instruct the user to share through the **QuickSight console**, or to run
  `cid-cmd share` **interactively** in a real terminal.

For reference, when upstream fixes this, the arguments would be:
`--dashboard-id <id> --share-method <account|user|folder>`; `user` requires
`--quicksight-user`; `folder` requires `--folder-method <new|existing>` then `--folder-name`
(new) or `--folder-id` (existing); `account` takes none of those.

(Summarized from `TODO.md`.)

## Other cid-cmd subcommands

Beyond deploy/update/delete/status/migrate/share, `cid-cmd` has more subcommands a user can
run manually. Full how-tos with real flags and safety framing are in
[references/cid-cmd-subcommands.md](./references/cid-cmd-subcommands.md). Highlights:

- **Read-only:** `open` (print a dashboard's QuickSight URL); `export` (publish an analysis to
  a reusable YAML — read-only in `definition` mode, but mutating when it creates a QuickSight
  `template`, so confirm that path).
- **Mutating — confirm first:** `refresh` (SPICE dataset refresh — consumes billable
  ingestion); `init-qs` (**activate QuickSight Enterprise — billable**; this is how you satisfy
  the QuickSight prerequisite); `create-cur-table` / `create-cur-proxy` (stand up or translate
  a CUR Athena table/view without the Data Exports stack).
- **Account maps:** `map` builds the `account_map` Athena view and `csv2view` turns a CSV into
  a view — `map` is how you *generate* the account map that
  `deploy --account-map-source csv --account-map-file <csv>` later consumes. Both are mutating.
- **Destructive — handle with care:**
  - `cleanup` deletes unused QuickSight datasets / Athena views. **Destructive — confirm.**
  - `teardown` deletes **ALL** CID assets. Upstream's own docstring says *"VERY DANGEROUS. DO
    NOT USE THIS COMMAND."* **Refuse by default** — run only on explicit, unambiguous user
    direction that names `teardown` specifically; otherwise steer to targeted `delete` or
    `cleanup`.

## Infrastructure via CloudFormation

Use direct `aws cloudformation deploy` / `create-stack` / `update-stack` with
`--capabilities CAPABILITY_NAMED_IAM` and the public template URLs. All exact URLs, pinned
versions, per-stack parameters, default stack names, and region constraints are in
[references/cloudformation-templates.md](./references/cloudformation-templates.md). Mutating —
confirm first.

Stacks you can drive:

- **Data Exports** (CUR 2.0 / FOCUS / Cost Optimization Hub) — run in the
  destination/data-collection account. Params: `DestinationAccountId`, `ManageCUR2`,
  `ManageFOCUS`, `ManageCOH` (`yes|no`), optional `SourceAccountIds`.
- **Legacy CUR 1.0 aggregation** — prefer Data Exports for new setups. Params:
  `DestinationAccountId`, optional `SourceAccountIds`.
- **Data Collection Step 1 — read-permissions stack** in the Organizations **MANAGEMENT**
  account. Params: `DataCollectionAccountID`, `OrganizationalUnitIds`, and one
  `IncludeXModule=yes` per module that needs a cross-account role (passing a no-role module
  here is an error).
- **Data Collection Step 2 — the pipeline** in a dedicated **DATA COLLECTION** account.
  Params: `ManagementAccountID`, optional `RegionsInScope`, and one `IncludeXModule=yes` per
  enabled module. Must be in one of the **18 supported regions**; pricing has no separate
  toggle (collected automatically with inventory / RDS / EUC utilization). The exact
  module-key → CFN-parameter mapping and which dashboards each module powers are in
  [references/data-collection-modules.md](./references/data-collection-modules.md).
- **CFN dashboard deploy (`cid-cfn.yml`)** — an alternative to `cid-cmd deploy`, limited to 5
  dashboards (CUDOS v5, Cost Intelligence, KPI, TAO, Compute Optimizer). Requires the
  QuickSight prerequisites. Params per the reference file.

Check and manage stacks:

- Status: `aws cloudformation describe-stacks --stack-name <name> ...` (pull recent
  `*_FAILED` stack events to diagnose a failure).
- List: `aws cloudformation list-stacks ...` (filter by a `cid` prefix).
- Delete: `aws cloudformation delete-stack --stack-name <name> ...` — **destructive**, confirm
  first.

## Workflows

### Foundational dashboards (CUDOS v5, Cost Intelligence, KPI — CUR-only)

1. Ensure a CUR 2.0 export exists — deploy the Data Exports stack, or reuse an existing one.
2. `cid-cmd ... deploy --dashboard-id <id>`.

### Advanced dashboards (TAO, Compute Optimizer, Graviton, Health Events, etc.)

Order matters:

1. Deploy the **Data Collection read-permissions** stack in the Organizations **Management**
   account (fans a read role across OUs via service-managed StackSets — needs Organizations
   trusted access).
2. Deploy the **Data Collection** pipeline stack in a dedicated **Data Collection** account,
   with the modules the dashboard needs enabled (see
   [references/data-collection-modules.md](./references/data-collection-modules.md)).
3. After the pipeline has collected data at least once, deploy the dashboard
   (`cid-cmd ... deploy --dashboard-id <id>`).

### Two ways to deploy a dashboard

- **`cid-cmd deploy`** — the full catalog, driven interactively-free with explicit flags.
- **CFN path (`cid-cfn.yml`)** — infrastructure-as-code, one tracked stack, but only 5
  dashboards.

## Gotchas and troubleshooting

- **"Unknown dashboard-id" on a catalog dashboard.** cid-cmd fetches its extended catalog from
  `raw.githubusercontent.com` on every run. If that fetch is blocked, cid-cmd silently falls
  back to its ~6 bundled dashboards and catalog-only dashboards (CORA, TAO, Graviton, Health
  Events, etc.) fail with "unknown dashboard-id" even though cid-cmd otherwise works. Check
  GitHub reachability first.
- **The two most common macOS setup failures (both on python.org framework builds):**
  - **`cid-cmd: command not found` even after a successful `pip install`.** The console
    script installed to the framework bin dir
    (`/Library/Frameworks/Python.framework/Versions/<X.Y>/bin/cid-cmd`), which is often not on
    `PATH`. Do not conclude it failed to install — LOCATE it (see the Prerequisites
    "Locate `cid-cmd`" rule) and invoke it by full path.
  - **`CERTIFICATE_VERIFY_FAILED` on every HTTPS call.** python.org framework builds can ship
    without a CA bundle, so every outbound HTTPS call (the GitHub catalog fetch and AWS API
    calls) fails. It looks like a network problem but is a local Python issue. Fix:
    `pip install certifi` and run `/Applications/Python*/Install Certificates.command` once.
- **`cid.log` in the project folder.** cid-cmd writes `cid.log` to the current directory. Run
  it from a scratch temp directory.
- **Credential errors — tell them apart from the stdout text:**
  - profile not found → check `aws configure list-profiles`.
  - `InvalidClientTokenId` / expired security token → re-authenticate (e.g. `aws sso login`).
  - `UnrecognizedClientException` → the access key/secret are wrong.
  - `AccessDenied` / "not authorized" → credentials are valid but the IAM policy lacks
    permission for the action.
- **Data Collection region limit.** The Data Collection stacks deploy only in the 18 supported
  regions (see the templates reference); other regions are refused with a clear message.

## Reference files

Load the one you need:

- [references/dashboard-catalog.md](./references/dashboard-catalog.md) — pick a dashboard or
  resolve a friendly-name alias to a canonical id; see which dashboards need Data Collection.
- [references/data-collection-modules.md](./references/data-collection-modules.md) — choose
  Data Collection modules and map each to its `IncludeXModule` CFN parameter.
- [references/cloudformation-templates.md](./references/cloudformation-templates.md) — exact
  template URLs, pinned versions, per-stack parameters (common path plus the full advanced
  parameter surface for Data Exports and Data Collection), and region constraints for any CFN
  stack.
- [references/cid-cmd-subcommands.md](./references/cid-cmd-subcommands.md) — the other
  `cid-cmd` subcommands the lifecycle sections above do not cover: `open`, `export`,
  `refresh`, `map`, `csv2view`, `init-qs`, `create-cur-table`, `create-cur-proxy`, `cleanup`,
  and `teardown`, each with real flags and the same safety framing.
- [COVERAGE.md](./COVERAGE.md) — the capability audit: what the skill covers vs. the full
  upstream manual surface, with the upstream subcommands/parameters cited per row.
