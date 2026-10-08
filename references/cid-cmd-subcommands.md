# Other `cid-cmd` subcommands

The `cid-cmd` subcommands the former MCP server never wrapped — and therefore the lifecycle
sections of SKILL.md do not cover — but which a user can run manually upstream. One concise
how-to each. Flags are taken verbatim from upstream `cid/cli.py` (cid-cmd 4.4.17); do not
invent flags.

**These rules from SKILL.md apply to every subcommand below:**

- **Global flags** work on all of them: `--profile <profile>` (alias of `--profile_name`),
  `--region_name <region>`, `-y`/`--yes` (confirm-all / unattended), `-v`/`--verbose`.
- **Judging pass/fail:** exit code 0 is NOT proof of success. Scan stdout and treat the run
  as FAILED if any line matches `^(ERROR|CRITICAL) - ` OR the process exits non-zero. See
  "Judging success and failure of cid-cmd" in SKILL.md, including the one `status`
  picker-traceback exception.
- **`cid.log` + cwd:** run from a scratch temp directory; cid-cmd writes `cid.log` to the
  current directory.
- **Athena connection chain (verified live):** ANY subcommand that queries Athena —
  confirmed for `create-cur-proxy` and `map`, and true of `deploy`/`update` — needs the Athena
  connection parameters unattended or it prompts-then-aborts: at minimum
  `--athena-database <db>` and `--athena-workgroup <wg-with-a-result-location>` (use the `CID`
  workgroup, not `primary`), plus `--cur-table-name <view>` for the CUR-based ones. Supply
  them up front. See "Required parameters for unattended deploy/update" in SKILL.md.
- **Close stdin on anything that might open the interactive picker:** append `< /dev/null`
  (as with `status`) so a subcommand cannot block on a menu in a TTY shell.
- **Safety:** mutating subcommands below are **real and billable in the user's AWS account**.
  You MUST get explicit user confirmation first, state the action is real/billable (and, where
  noted, destructive/irreversible), and prefer the read-only ones when the user is only
  inspecting. This is prompt-enforced only — there is no code backstop.

---

## Read-only / low-risk

### `open` — open the dashboard's QuickSight URL
`cid-cmd ... open --dashboard-id <id> < /dev/null`

Opens the QuickSight URL for a deployed dashboard. Read-only. **Verified live: headless it
prints no usable URL** — it tries to launch a browser via the OS, which does nothing in an
agent/headless context, and emits nothing actionable to stdout. So do NOT rely on `open` to
*retrieve* a URL. **To get the URL non-interactively, use `status --dashboard-id <id>`
instead** — its output includes the `https://<region>.quicksight.aws.amazon.com/sn/dashboards/<id>`
line. Always pass `--dashboard-id` (omitting it triggers the interactive picker).

### `export` — publish an analysis/dashboard to a reusable YAML definition
```
cid-cmd ... export \
  [--analysis-id <id> | --analysis-name <name>] \
  [--dashboard-id <id>] \
  [--dashboard-export-method definition|template] \
  [--output <file.yaml>] \
  [--taxonomy <field,field,...>] \
  [--reader-account <account-id|*>] \
  [--export-known-datasets no|yes] \
  [--export-tables no|yes] \
  [--one-file no|yes] \
  [--template-id <id>] [--template-version <vX.Y.Z>] \
  [--category <name>]
```

Exports a deployed analysis/dashboard to a reusable CID resources YAML — the upstream way to
package a **custom** dashboard so it can be re-deployed (via `deploy --resources <file>`).
Identify the source with `--analysis-id` (open the analysis in the browser and copy the id
from the URL) or `--analysis-name`.

- `--dashboard-export-method definition` pulls the JSON definition of the analysis (local
  only). `--dashboard-export-method template` **creates a QuickSight Template** in the
  account — that is a real mutating action, so **confirm before using `template`**.
- `--reader-account` grants read access to another account id (or `*`) — treat as sharing;
  confirm.
- `--export-tables yes` includes Athena tables (default is views only);
  `--export-known-datasets yes` includes datasets already in the resources file.
- `--output <file.yaml>`: if the file exists it is read for defaults and overwritten.

The `definition` method with a local `--output` is effectively read-only against AWS; the
`template` method and `--reader-account` are not — confirm those.

---

## Mutating — confirm first

### `refresh` — trigger a SPICE dataset refresh
`cid-cmd ... refresh [--dashboard-id <id>]`

Triggers a SPICE ingestion/refresh for the datasets behind a dashboard. **Mutating** — it
consumes SPICE ingestion capacity (which is billable) and starts refresh jobs. Confirm first.
Pass `--dashboard-id <id>` so it does not fall into the interactive picker.

### `map` — build the `account_map` Athena view
```
cid-cmd ... map [--view-name <name>] [--simple] [--file <accounts.csv>] [--database <db>]
```

Builds the `account_map` Athena view (default view name `account_map`) that maps account IDs
to friendly names / taxonomy dimensions. **This is how you GENERATE the account map** that
`deploy --account-map-source` later consumes — `deploy` reads the map, `map` creates it.
**Mutating** (creates/overwrites an Athena view). Options:

- `--view-name <name>` — output view name (default `account_map`).
- `--simple` — legacy simple account mapping mode (backwards compatibility).
- `--file <accounts.csv>` — path to a CSV of taxonomy dimensions (must exist).
- `--database <db>` — source database containing `organization_data` (skips discovery).

### `csv2view` — generate an Athena view from a CSV
`cid-cmd ... csv2view --input <file.csv> --name <AthenaViewName>`

Generates an Athena view from an arbitrary CSV (`--input` = the CSV file, `--name` = the
Athena view name to create). **Mutating** (creates an Athena view). Use it to turn a
spreadsheet of lookup/taxonomy data into a queryable view. (The command is **`csv2view`** —
there is no `csv2labels`.)

### `init-qs` — activate QuickSight Enterprise non-interactively
```
cid-cmd ... init-qs \
  --enable-quicksight-enterprise yes \
  --account-name <unique-qs-account-name> \
  --notification-email <email>
```

Activates **Amazon QuickSight Enterprise Edition** without the console wizard. **Mutating AND
billable** — QuickSight Enterprise has an ongoing cost — so confirm emphatically and make the
cost explicit before running. This is the one command that satisfies the hard "QuickSight
Enterprise with SPICE must exist first" prerequisite in SKILL.md, so reach for it when a user
has no QuickSight set up yet.

- `--enable-quicksight-enterprise yes` — confirm activation.
- `--account-name <name>` — a QuickSight account name that must be unique across all AWS.
- `--notification-email <email>` — email for QuickSight notifications.

### `create-cur-table` — create the CUR Athena table from an existing S3 CUR path
```
cid-cmd ... create-cur-table \
  --view-cur-location s3://<bucket>/<path> \
  [--crawler-role <name-or-arn>]
```

Creates the CUR Athena table directly from an existing S3 CUR location, **without** the Data
Exports CloudFormation stack. **Mutating** (creates a Glue crawler/role and Athena table).
Upstream supports two CUR path shapes only: `s3://{bucket}/cur` and
`s3://{bucket}/{prefix}/{cur_name}/{cur_name}`. Use it when a CUR already lands in S3 and the
user just needs it queryable in Athena. `--crawler-role` is the name or ARN of the crawler
role.

### `create-cur-proxy` — Athena proxy view translating CUR1 ↔ CUR2
```
cid-cmd ... create-cur-proxy \
  --cur-version 1|2 \
  [--fields <field,field,...>] \
  [--cur-table-name <name>] [--cur-database <db>] [--athena-database <db>]
```

Creates an Athena proxy **view** that transforms CUR1→CUR2 or CUR2→CUR1, so a dashboard built
for one CUR version can read the other. **Mutating** (creates an Athena view). This is a
**lighter alternative to a full CUR 2.0 migration** (`update --force --recursive`): it does
not rebuild datasets/views, it just presents the other shape. `--cur-version` is the *target*
version; `--fields` adds extra CUR fields; the `--cur-table-name` / `--cur-database` locate
the existing CUR and `--athena-database` is where the proxy view is created.

### `cleanup` — delete unused QuickSight datasets / Athena views
`cid-cmd ... cleanup`

Deletes QuickSight datasets and Athena views that are **not** tied to any CID-managed
dashboard. **Destructive — confirm first.** It removes resources, and anything it deletes is
gone. Warn the user it can remove datasets/views they built by hand if those are not attached
to a managed dashboard. Prefer a targeted `delete --dashboard-id <id>` when the user only
wants to remove one dashboard's resources.

---

## Extremely destructive — refuse by default

### `teardown` — delete ALL CID assets
`cid-cmd ... teardown`

Deletes **every** CID asset. Upstream's own docstring for this command says, verbatim:
**"THIS IS VERY DANGEROUS. DO NOT USE THIS COMMAND."**

- **Default to refusing.** Do NOT run `teardown` on a vague request like "clean this up" or
  "remove the dashboards."
- Run it **only** on explicit, unambiguous user direction that names **`teardown`
  specifically** and acknowledges it deletes all CID assets irreversibly.
- Even then, state plainly that this is irreversible and wipes everything CID created, and get
  a second explicit confirmation.
- Offer the **targeted alternatives first**: `delete --dashboard-id <id>` for a single
  dashboard, or `cleanup` for unused datasets/views. Steer the user to those unless they truly
  want a full wipe.
