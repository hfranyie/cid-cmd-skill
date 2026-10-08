# cid-cmd-skill

An **Agent Skill** for deploying and managing [AWS Cloud Intelligence Dashboards
(CID)](https://github.com/aws-solutions-library-samples/cloud-intelligence-dashboards-framework)
— CUDOS, Cost Intelligence, KPI, Trusted Advisor (TAO), CORA, Graviton, Health Events,
FOCUS, and the rest of the CID catalog — by chatting with an AI agent.

Instead of exposing CID through a running MCP server, this packages the knowledge as a
portable **`SKILL.md`** that an agent (Kiro, Claude, or any system that reads a skill /
instruction file) loads on demand and uses to drive the public tools directly:

- the **`cid-cmd`** CLI (`pip install cid-cmd`) for dashboard lifecycle, and
- public **CloudFormation** templates for the data-exports and data-collection infrastructure,

all with the user's **own AWS credentials**. No special account, no vendored credential
broker, no internal dependency.

## Why

Deploying CID normally means installing a CLI, reading CloudFormation parameter docs, and
clicking through the console. This skill lets you just *ask*:

> *"Deploy CUDOS in my account."*
> *"Is my CUDOS dashboard up to date?"*
> *"Set up CUR 2.0 data exports from account A into my data-collection account."*
> *"Delete the KPI dashboard."*

The agent loads the skill, resolves the friendly name to the right dashboard id, assembles
the exact `cid-cmd` / CloudFormation invocation (including the fiddly Athena
database/workgroup/datasource/CUR-table parameter chain that trips up unattended runs),
runs it, and reads the result correctly — including CID's known quirks (the `status` command
that prints a valid result and then throws an interactive-picker traceback, the "exit 0 can
still be a failure" log convention, the off-PATH `cid-cmd` binary on macOS, and more). Those
hard-won operational details are baked into the skill, so the agent gets them right the first
time.

## What's here

| File | Purpose |
|---|---|
| [`SKILL.md`](./SKILL.md) | The skill itself — frontmatter (`name`, `description`) + agent instructions covering auth, prerequisites, safety/confirmation rules, read-only ops, deploy/update/delete/migrate, the CloudFormation data-exports & data-collection flows, and troubleshooting. |
| [`references/dashboard-catalog.md`](./references/dashboard-catalog.md) | Full dashboard catalog + friendly-name → id aliases. |
| [`references/data-collection-modules.md`](./references/data-collection-modules.md) | The 24 Data Collection modules → CloudFormation parameter mappings. |
| [`references/cloudformation-templates.md`](./references/cloudformation-templates.md) | Template bucket, pinned versions, exact URLs, per-stack parameters, supported regions, and live-validated CFN behaviors/gotchas. |
| [`COVERAGE.md`](./COVERAGE.md) | Capability matrix: does the skill cover everything the upstream repos let a user do? |
| [`VALIDATION.md`](./VALIDATION.md) | What has been executed against live AWS vs. documented-only vs. out-of-scope, with evidence. |

## Using it

- **Kiro:** copy the skill folder into `~/.kiro/skills/cid-cmd-skill/` (or a workspace
  `.kiro/skills/`); it activates on relevance. Then just chat.
- **Claude:** the same `SKILL.md` + frontmatter format works as a Claude Agent Skill.
- **Other agents:** the `SKILL.md` body is plain markdown — load it as a rules / instruction
  file (e.g. `AGENTS.md`).

```bash
# install for Kiro
git clone https://github.com/hfranyie/cid-cmd-skill.git
cp -r cid-cmd-skill ~/.kiro/skills/cid-cmd-skill
# make sure cid-cmd and AWS creds are available
pip install cid-cmd
# then, in your agent:  "check the status of CUDOS and tell me if it needs updating"
```

### Example interaction

> **You:** check the status of CUDOS and tell me if it needs updating
>
> **Agent:** *(loads the skill, locates `cid-cmd`, resolves "CUDOS" → `cudos-v5`, runs the
> read-only status check, reads past the known picker traceback)*
> CUDOS Dashboard v5 is **up to date** — version `v5.9.1 (latest)`, health **healthy**. No
> update needed. (Note: datasets show 0 rows, which points to the underlying CUR data being
> empty rather than a dashboard problem.)

This exact exchange was run against live AWS with a fresh agent that had only the skill
loaded — see [`VALIDATION.md`](./VALIDATION.md).

## Prerequisites (summarized — see `SKILL.md`)

- Python 3.10+ and `pip install cid-cmd`
- Your own AWS credentials (profile / SSO / env / instance role)
- QuickSight Enterprise Edition + SPICE active in the target region before deploying dashboards
- Outbound HTTPS to `raw.githubusercontent.com` (cid-cmd fetches its extended catalog)

## Safety

Deploying CID creates real, billable AWS resources. Unlike a code-enforced MCP server, a
skill's confirm-before-mutating gates are **prompt-enforced** — the skill instructs the agent
to confirm before any deploy/update/delete and to prefer read-only operations when inspecting.
Review changes before approving them.

## Status

Resource-lifecycle operations (dashboard deploy/update/delete/status, CloudFormation
data-exports deploy/upgrade, multi-account source→destination wiring) are validated against
live AWS — see `VALIDATION.md`. End-to-end **data flow** (dashboards populated with real cost
data) and the **data-collection pipeline / advanced dashboards** were not live-validated in
the test sandboxes (they require accounts with real spend and an Organizations management
account); they are documented and audited. See `VALIDATION.md` for the honest boundary.

## License

MIT-0. See [LICENSE](./LICENSE).

## Related

- [Cloud Intelligence Dashboards Framework (`cid-cmd`)](https://github.com/aws-solutions-library-samples/cloud-intelligence-dashboards-framework)
- [CID Data Collection (CloudFormation templates)](https://github.com/aws-solutions-library-samples/cloud-intelligence-dashboards-data-collection)
- [`cid-cmd` on PyPI](https://pypi.org/project/cid-cmd/)
