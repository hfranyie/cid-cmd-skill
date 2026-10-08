# CID skill — live validation status

Separate from `COVERAGE.md` (which tracks whether the skill *documents* each upstream
capability), this file tracks whether each capability has been **executed against live AWS**
and the skill's guidance confirmed correct in practice. "Documented" is necessary but not
sufficient — every lane we have actually run has exposed a real gap, so an un-run lane is not
yet trustworthy.

## Test environment

- `<DEST-ACCT>` — an Isengard **member** account in an AWS Organization `<ORG-ID>`
  (FeatureSet `ALL`). The Org **management** account `<MGMT-ACCT>` was not available to us (**no
  credentials**). Region `us-east-1`. QuickSight Enterprise + SPICE already active; a CUR 2.0
  export (`cid_cur` / `cur2_view`) already present. Credentials were temporary STS and expired
  mid-session. (Real account IDs redacted; `<DEST-ACCT>` = data-collection/destination,
  `<SOURCE-ACCT>` = source, `<MGMT-ACCT>` = Org management.)

## Legend

- **Live-validated** — executed end-to-end against this account; skill guidance confirmed (and
  any gap found was fixed in the skill).
- **Pending** — testable here, not yet run.
- **Out of scope (member account)** — requires the Org management account / an org-wide
  StackSet we cannot run from this member-account sandbox; validated only by template-URL
  resolution + parameter audit, NOT by live deploy.

## Status

| Capability | State | Evidence / notes |
|---|---|---|
| `status` (overall + `--dashboard-id`) | **Live-validated** | Ran on `cudos-v5`. Found + fixed: off-PATH binary locate; exit-nonzero-but-success picker traceband; and the **picker *hang*** in TTY shells → skill now requires `< /dev/null`. |
| `update` (dashboard version) | **Live-validated** | `cudos-v5` v5.7.0→v5.9.1, confirmed up to date. Found + fixed: required Athena param chain (`--athena-database`/`--athena-workgroup`/datasource/`--cur-table-name`), `--timezone` for new Data Prep migration, exit-0-can-still-be-failure rule. |
| `delete` | **Live-validated** | Deleted `cudos-v5`; shared dependencies (`account_map`, `summary_view`) correctly preserved. Needs no Athena params. |
| `deploy` (from scratch, by hand) | **Live-validated** | Recreated `cudos-v5` healthy at v5.9.1. Found + fixed: full param chain; `--recursive` is **update-only** (deploy rejects it); stale-CUR-view (`cur2_view` 73→130 cols) repair via `CREATE OR REPLACE VIEW`. |
| `deploy` **via chat (fresh skill-only agent)** | **Pending** | Target: net-new `trends-dashboard`. Tests whether a fresh agent assembles the param chain unaided. |
| `status` **via chat (fresh skill-only agent)** | **Live-validated** | Fresh agent, MCP tools disabled, skill-only: correctly reported CUDOS up to date. Surfaced the picker-hang fix above. |
| `create-cur-proxy` | **Live-validated** | Created `cur1_proxy` view in `cid_cur` (exit 0, view present). Needs the Athena chain incl. `--athena-workgroup CID`. |
| `map` (`--simple`) | **Live-validated** | Built `account_map` (exit 0, detected CUR2 correctly). Needs the Athena chain. |
| `open` | **Live-validated** | Runs/authenticates but prints no usable URL headless — use `status --dashboard-id` to get a URL. Skill corrected. |
| Remaining subcommands: `refresh`, `csv2view`, `create-cur-table`, `export`, `cleanup` | **Pending** | Cheap, testable here. `cleanup` destructive — confirm. |
| `teardown` | **Won't test** | Deletes ALL CID assets; skill is refuse-by-default. Not worth exercising live. |
| CFN stack create / update / status / delete / list mechanics | **Live-validated** | Exercised across both accounts: create-stack, update-stack (`UsePreviousValue` for unchanged params), describe-by-ARN after rollback, event timeline, delete, list-stacks — all behaved as the skill describes. |
| `deploy_data_exports` — destination side | **Live-validated (pre-existing) + update** | Account `<DEST-ACCT>` already had `CID-DataExports-Destination` (CREATE_COMPLETE). Updated its `SourceAccountIds` → `<SOURCE-ACCT>` to `UPDATE_COMPLETE`; confirmed the destination **bucket policy** then granted the source replication access. A *second* destination stack is correctly refused (duplicate CFN export `cid-DataExports-Database`). |
| `deploy_data_exports` — destination template upgrade (v0.9.0 → v0.12.0) | **Live-validated** | The destination was running the OLD `v0.9.0` template (replication-only bucket policy, no `s3:PutObject`/`S3BucketOwner`). `update-stack --template-url <0.12.0>` upgraded it in place to `UPDATE_COMPLETE`; the bucket policy gained the `EnableAWSDataExportsToWriteToS3` `s3:PutObject` grant for `bcm-data-exports.amazonaws.com`. |
| `deploy_data_exports` — source side (multi-account), end-to-end | **Live-validated** | ROOT-CAUSED: the earlier `S3 bucket permission validation failed` was NOT a race — it was a **version mismatch** (v0.12.0 source export writes directly to the destination bucket and needs the `s3:PutObject` grant that only the v0.12.0 destination policy provides; the v0.9.0 destination lacked it). After upgrading the destination to v0.12.0, the source export stack in `<SOURCE-ACCT>` reached **CREATE_COMPLETE** — `LocalCUR2viaCFN` (AWS::BCMDataExports::Export), `SourceS3`, and the analytics Lambda/role all created. Multi-account source→destination export fully deployed. Correct ordering + fix documented in cloudformation-templates.md. |
| `deploy_dashboards_via_cloudformation` (`cid-cfn.yml`) | **Pending** | Testable here. Requires QuickSight prereqs (present). |
| CFN stack status / list / delete | **Pending** | Testable; delete needed to clean up the two above. |
| `deploy_data_collection_permissions` (step 1) | **Out of scope (member account)** | Must run in management account `<MGMT-ACCT>` (no creds) and fans out an org-wide StackSet. Template URL resolves (HTTP 200); parameters audited. Not live-deployed here. |
| `deploy_data_collection` (step 2) | **Out of scope (member account)** | Depends on step 1 in the management account. Template URL resolves; parameters audited. Not live-deployed here. |
| Advanced dashboards (TAO, CORA, Graviton, Health Events, …) | **Out of scope (member account)** | Depend on the Data Collection pipeline above; data would be empty without it. Catalog entries audited; not live-deployed. |
