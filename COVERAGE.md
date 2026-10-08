# CID skill coverage audit

Does the `cid-cmd-skill` skill let a user do, by chatting with an agent, everything the two
upstream open-source projects let them do by hand? This is the capability matrix that
answers that question.

> **Status: gaps closed (skill v1.1.0).** This document was first written as a *pre-work
> audit* that found the gaps below, then the skill was extended to close them (commit
> `c8ac933`: `references/cid-cmd-subcommands.md` added for the 10 missing subcommands, and
> the advanced Data Exports / Data Collection CloudFormation parameters added to
> `references/cloudformation-templates.md`). The tables below now show the **post-fix** state;
> the "Note" column records what was originally missing and where it was added, so the matrix
> doubles as a record of the audit and its resolution.

**Audited against (shallow clones, read-only):**

- `cloud-intelligence-dashboards-framework` — the `cid-cmd` CLI. Command surface read from
  `cid/cli.py` (every click command) and the method bodies in `cid/common.py`. Upstream
  `cid/_version.py` is `4.4.17`, identical to the version the skill pins — so there is **no
  version drift** to reconcile; the gaps below are coverage gaps, not staleness.
- `cloud-intelligence-dashboards-data-collection` — the Data Exports + Data Collection
  CloudFormation templates. Parameter surface read from each template's own `Parameters:`
  block (`data-exports/deploy/data-exports-aggregation.yaml`,
  `data-collection/deploy/deploy-data-collection.yaml`,
  `data-collection/deploy/deploy-data-read-permissions.yaml`).

**Legend (post-fix state):** Full = a user can do it through the skill today with correct
guidance • Partial = possible but some options/behavior remain under-documented • None = still
not covered.

**Headline:** the skill now covers the full practical upstream surface. Beyond the dashboard
lifecycle the former MCP server wrapped (deploy/update/delete/status/migrate) and the three
main CFN stacks, the gap-closing commit added (1) the 10 `cid-cmd` subcommands the MCP server
never wrapped — `export`, `cleanup`, `teardown`, `open`, `refresh`, `map`, `csv2view`,
`init-qs`, `create-cur-table`, `create-cur-proxy` (now in
`references/cid-cmd-subcommands.md`) — and (2) the advanced CloudFormation parameter sets on
the Data Exports and Data Collection stacks (now in `references/cloudformation-templates.md`).
What remains un-exercised is live end-to-end testing against a real AWS account, which is a
separate human-gated step, not a documentation gap.

---

## 1. Dashboard lifecycle — `cid-cmd` subcommands

| Upstream capability (cli.py command → common.py method) | Skill coverage | Note |
|---|:---:|---|
| `deploy` (`Cid.deploy`) | **Full** | All allowlisted flags documented in SKILL.md "Deploy a dashboard". Missing only the upstream `--category` filter flag (minor; dashboard-id is the normal path). |
| `update` (`Cid.update`) | **Full** | `--force`, `--recursive`, `--on-drift`, `--theme`, `--currency`, `--rls`/`--rls-dataset-id` all documented. |
| `delete` (`Cid.delete`) | **Full** | `--dashboard-id`, `--athena-database`, destructive framing present. |
| `status` (`Cid.status`) | **Full** | Documented, and the post-status picker-traceback recognition rule is now stated explicitly in SKILL.md ("treat the status block above the trailing `Please set parameter <id>...` exception as SUCCESS"), matching the former MCP server's `_completed_status_before_headless_picker`. |
| `share` (`Cid.share`) | **Full (as "broken")** | Correctly documented as non-functional on 4.4.17: `share(ctx, dashboard_id)` drops all parsed `**kwargs`. Keep; re-verify the signature note. |
| migrate to CUR 2.0 (`update --force --recursive`) | **Full** | Documented as its own procedure with the override warning. |
| `export` (`Cid.export`) | **Full** | Added to `references/cid-cmd-subcommands.md` with the `--analysis-id`/`--dashboard-export-method definition\|template`/`--output`/`--taxonomy`/`--reader-account` surface and the "confirm before `template`/`--reader-account` (mutating/sharing)" framing. |
| `cleanup` (`Cid.cleanup`) | **Full** | Added, framed destructive ("confirm first"), with a steer to targeted `delete --dashboard-id` instead. |
| `teardown` (`Cid.teardown`) | **Full** | Added under an "extremely destructive — refuse by default" heading, quoting upstream's "VERY DANGEROUS. DO NOT USE" and requiring explicit, named, double confirmation. |
| `open` (`Cid.open`) | **Full** | Added as a read-only helper; pass `--dashboard-id` to avoid the interactive picker. |
| `refresh` (`Cid.refresh_datasets`) | **Full** | Added, framed mutating (consumes billable SPICE ingestion); confirm first. |
| `map` (`Cid.map`) | **Full** | Added with `--view-name`/`--simple`/`--file`/`--database`; documents that `map` GENERATES the account map that `deploy --account-map-source` later consumes. |
| `csv2view` (`Cid.csv2view`) | **Full** | Added (`--input`, `--name`); the audit's guessed `csv2labels` was corrected to the real `csv2view`. |
| `init-qs` (`Cid.init_qs`) | **Full** | Added (`--enable-quicksight-enterprise`/`--account-name`/`--notification-email`), framed mutating+billable; noted as the one command that can satisfy the QuickSight-Enterprise prerequisite. |
| `create-cur-table` (`Cid.create_cur_table`) | **Full** | Added (`--view-cur-location`, `--crawler-role`) incl. the two supported CUR S3 path shapes. |
| `create-cur-proxy` (`Cid.create_cur_proxy`) | **Full** | Added (`--cur-version`, `--fields`, locating params) as the lighter alternative to a full CUR 2.0 migration. |

## 2. Data Exports — `data-exports-aggregation.yaml`

Skill documents `DestinationAccountId`, `ManageCUR2`, `ManageFOCUS`, `ManageCOH`,
`SourceAccountIds`. The template's actual `Parameters:` block exposes considerably more.

| Upstream parameter | Skill coverage | Note |
|---|:---:|---|
| `DestinationAccountId`, `ManageCUR2`, `ManageFOCUS`, `ManageCOH`, `SourceAccountIds` | **Full** | Documented in cloudformation-templates.md. |
| `ManageCarbon` | **Full** | Added to cloudformation-templates.md — carbon-emissions export feeding the Sustainability dashboard. |
| `EnableSCAD` | **Full** | Added — Split Cost Allocation Data (container/ECS) toggle. |
| `EnableIAMPrincipalData` | **Full** | Added — IAM principal attribution data. |
| `CUR2TimeGranularity`, `FOCUSTimeGranularity` | **Full** | Added — HOURLY/DAILY/MONTHLY granularity choice. |
| `ResourcePrefix` | **Full** | Added, with the side-by-side / match-existing note. |
| `RolePath` | **Full** | Added — IAM role path for permission-boundary/SCP regimes. |
| `LakeFormationEnabled` | **Full** | Added — register data in Lake Formation. |
| `LegacyLocalBucket`, `SecondaryDestinationBucket` | **Full** | Added — reuse an existing bucket / replicate to a second destination. |
| `AddScheduleForBlockingWrite`, `DisableWriteCronSchedule`, `EnableWriteCronSchedule`, `IsSourceAccount`, `DeployDataExport`, `DeployWriteBlocker`, `DataExports` | **Full** | Added — advanced multi-account write-blocker / scheduling / source-vs-destination controls. |

## 3. Data Collection pipeline — `deploy-data-collection.yaml` + `deploy-data-read-permissions.yaml`

The 24-module `IncludeXModule` matrix is **Full** and verified against both templates'
`Parameters:` blocks (every `IncludeXModule` present; `KiroSourceBuckets`, `EUCAccountIDs`,
and the no-cross-account-role rule all correct). The gaps are the **non-module stack
parameters**.

| Upstream parameter (stack) | Skill coverage | Note |
|---|:---:|---|
| All 24 `IncludeXModule` toggles + `EUCAccountIDs` + `KiroSourceBuckets` | **Full** | data-collection-modules.md, verified against the template. |
| `ManagementAccountID`, `RegionsInScope` (collection stack) | **Full** | Documented. |
| `DataCollectionAccountID`, `OrganizationalUnitIds` (permissions stack) | **Full** | Documented. |
| `Schedule`, `ScheduleFrequent` (collection) | **Full** | Added — EventBridge cadence for the frequent/less-frequent collection modules. |
| `DatabaseName` (collection) | **Full** | Added — target Athena/Glue database name. |
| `ResourcePrefix` (both stacks) | **Full** | Added, with the "must match across stacks" warning. |
| `MultiAccountRoleName` (both stacks) | **Full** | Added, explicitly flagged as the cross-account role name that **MUST match** between the permissions stack and the collection stack. |
| `ManagementAccountRole` (both stacks) | **Full** | Added — role assumed in the management account. |
| `DataBucketsKmsKeysArns` (collection) | **Full** | Added — KMS keys for encrypted data buckets. |
| `AllowModuleReadInMgmt` (permissions) | **Full** | Added — also collect management-account data, not just linked accounts. |
| `CFNSourceBucket` (both) | **Full** | Added — override the template/Lambda source bucket (advanced/air-gapped). |

## 4. Other upstream capabilities

| Capability | Where | Skill coverage | Note |
|---|---|:---:|---|
| Legacy CUR 1.0 aggregation stack | `cur-aggregation.yaml` | **Full** | Documented with a "prefer Data Exports" steer. |
| CFN-native dashboard deploy | `cid-cfn.yml` | **Full** | 5-dashboard limitation and all params documented. |
| 18-region Data Collection limit | `RegionMap` | **Full** | Documented. |
| 2-region CFN-native CUR fallback | template logic | **Full** | Documented, including the slow-Lambda-path explanation. |
| Judging pass/fail (exit 0 on failure) | `cid_cmd_runner.py` `_FAILURE_LOG_RE` | **Full** | `ERROR -`/`CRITICAL -` rule documented. |
| Locating the `cid-cmd` executable off-PATH | `cid_cmd_runner.py` `resolve_cid_cmd_path` | **Full** | Added to SKILL.md Prerequisites: a "do NOT assume it is on PATH" locate rule (venv `bin/` → `which` → sysconfig scripts dir), mirroring the former runner, with the `/Library/Frameworks/.../bin/cid-cmd` reality observed live. |
| macOS `CERTIFICATE_VERIFY_FAILED` fix | README/known issue | **Full** | Documented in Gotchas. |
| GitHub catalog fetch dependency | `cid/common.py` catalog URLs | **Full** | Documented. |
| `cid.log` written to cwd → run from scratch dir | `cid_cmd_runner.py` | **Full** | Documented. |

---

## Verdict

**Documentation coverage is complete (skill v1.1.0).** The gaps this audit originally found
are closed: the ~10 `cid-cmd` subcommands the former MCP server never wrapped are now in
`references/cid-cmd-subcommands.md` with correct safety framing, the advanced CFN parameter
sets on both the Data Exports and Data Collection stacks are in
`references/cloudformation-templates.md` (including the `MultiAccountRoleName` /
`ResourcePrefix` cross-stack coupling), the `status` picker-traceback recognition rule and the
exit-code-is-not-success failure rule are stated in SKILL.md, and the off-PATH
`cid-cmd`-locate guidance is in Prerequisites. By the "can the user do everything the repos
allow, via the skill?" measure, the skill now covers the full practical upstream manual
surface.

**What is still unproven is execution, not coverage.** No end-to-end run has driven the skill
conversationally against a live AWS account to confirm an agent actually produces the right
commands and recovers from the real-world snags (the `status` traceback, the off-PATH binary,
the GitHub catalog fetch). That live, human-gated validation — read-only first, then a
confirmed mutating action in a sandbox — is the remaining step and is tracked outside this
document.
