# busbar-actions/sf-action-sandbox

Run a single **declarative Salesforce CLI action** inside a capability-based
sandbox — kernel-enforced confinement (Linux [Landlock](https://docs.kernel.org/userspace-api/landlock.html))
on GitHub-hosted runners — with a full audit trail.

## What it does

1. Takes a declarative action (`SfAction` JSON) describing *what* to do.
2. Resolves the action's **minimal capability manifest** — exactly which
   filesystem paths, network hosts, and credentials the `sf` invocation needs,
   derived from the action's inherent risk level.
3. Spawns `sf` confined to that manifest. On Linux runners the spawned process
   is irreversibly restricted via Landlock *before* `exec` (a `pre_exec` hook in
   the forked child); the parent runner stays unconfined.
4. Parses the structured result and persists a **tamper-evident audit record**
   (Blake3 manifest hash, granted capabilities, outcome) as a workflow artifact.

The sandbox is built on the [`nono`](https://docs.rs/nono) capability runtime.
GitHub-hosted runners are Linux 5.13+, so `mode: enforce` applies real Landlock
restrictions there.

> **Prerequisite:** this action wraps the Salesforce CLI, so `sf` must be on
> `PATH` in the job (e.g. `npm install -g @salesforce/cli`) and the org must
> already be authenticated (pair with `busbar-actions/sf-org-create`).

## Inputs

| Input | Default | Description |
|---|---|---|
| `action` (required) | — | The declarative action as `SfAction` JSON. |
| `mode` | `enforce` | `enforce` (kernel-confined), `audit` (run + log policy), or `disabled`. |
| `working-dir` | `` | Working directory for the `sf` invocation. |
| `output` | `` | Path to write the structured `sf` result JSON. |
| `audit-out` | `.busbar/sandbox-audit.json` | Path to write the JSON audit log. |
| `version` | `latest` | `sf-action-sandbox` binary release tag. |
| `binary-repo` | `busbar-actions/actions-dist` | Where to fetch the binary. |

## Outputs

| Output | Description |
|---|---|
| `success` | `"true"` when the action executed successfully. |
| `duration-ms` | Wall-clock duration of the `sf` invocation. |
| `risk-level` | Inherent risk of the action (`low`/`medium`/`high`/`critical`). |
| `sandbox-enforced` | `"true"` when kernel confinement was applied. |
| `audit-path` | Path to the JSON audit log written for this run. |
| `result-path` | Path to the structured result JSON (when `output` is set). |

## Action JSON

Actions are tagged on the `action` field. Examples:

```json
{ "action": "data_query", "target_org": "my-org", "soql": "SELECT Id FROM Account" }
```
```json
{ "action": "project_deploy", "target_org": "staging", "source_dir": "force-app", "test_level": "RunLocalTests" }
```
```json
{ "action": "org_delete_scratch", "target_org": "pr-123" }
```

Risk levels drive the policy: read-only queries are `low` (network restricted to
Salesforce hosts), deploys are `high` (snapshot recommended), anonymous Apex and
scratch-org deletion are `critical` (approval-gated). See the crate's
[`SfAction`](../../busbar-extensions/crates/sf-action-sandbox/src/action.rs)
for the full vocabulary.

## Example

```yaml
permissions:
  contents: read

jobs:
  query:
    runs-on: ubuntu-latest        # Linux → Landlock enforcement
    environment: devhub-pbo-scratch
    steps:
      - name: Install Salesforce CLI
        run: npm install -g @salesforce/cli

      # … authenticate the org (e.g. busbar-actions/sf-org-create) …

      - uses: busbar-actions/sf-action-sandbox@v1
        id: q
        with:
          mode: enforce
          action: |
            {
              "action": "data_query",
              "target_org": "my-org",
              "soql": "SELECT Id, Name FROM Account LIMIT 10"
            }
          output: .busbar/accounts.json

      - name: Upload audit trail
        uses: actions/upload-artifact@v4
        with:
          name: sandbox-audit
          path: ${{ steps.q.outputs.audit-path }}
```

## Local / non-Linux behavior

`mode: enforce` requires a Unix kernel sandbox (Linux Landlock or macOS
Seatbelt). On unsupported platforms `enforce` fails fast rather than running
unconfined — use `mode: audit` to execute while only logging the policy that
*would* be enforced.
