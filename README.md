> [!WARNING]
> **`busbar-actions` is under heavy active development — expect breaking changes.**
> These repositories are public, but **not ready for use yet** — please don't depend on them.
> A pilot is starting soon: **[star and watch the busbar-actions organization](https://github.com/busbar-actions)** for the launch of Discussions and the pilot announcement.

# busbar-actions/sf-cli

Run a single **declarative Salesforce CLI action** inside a capability-based
sandbox — kernel-enforced confinement (Linux [Landlock](https://docs.kernel.org/userspace-api/landlock.html))
on GitHub-hosted runners — with a full audit trail.

The action first installs a version-pinned `@salesforce/cli` into a controlled
prefix, then executes the declarative action against that managed binary. You do
not need to preinstall `sf` on `PATH`.

## Authentication (zero stored credentials)

The `sf-cli` binary **mints its own short-lived Salesforce session in-process**
from the runner's GitHub **OIDC** id-token (exchanged at a Salesforce org with
Busbar installed and trust established) and **holds that token in the supervisor's
own memory**. It logs the managed `sf` CLI into the org **through an in-process
injecting proxy** using a **placeholder** token; the proxy swaps in the real
`Authorization` header **server-side**, so the managed `sf` **never holds the real
bearer token** — the `~/.sf` entry that lands on disk carries only the placeholder.
The session is **revoked and zeroized** at job end. There is **no `org-auth`
handoff step**, and **no token is written to `GITHUB_ENV`** or passed as an action
input/output.

To use it, the calling job must:

1. Grant `permissions: id-token: write` (so GitHub injects the OIDC endpoint).
2. Set `target-instance` to the Busbar-equipped org's instance URL.

> ### How the managed CLI stays token-free
>
> Every other Busbar action binary keeps the minted token **only in zeroizing
> memory** and calls the Salesforce API directly. `sf-cli` is different: it shells
> out to the managed `@salesforce/cli`, which persists auth to `~/.sf`. Rather than
> hand `sf` the real token, the run is confined to a **busbar-sandbox** session
> whose network egress is kernel-forced (`ProxyOnly`) through the supervisor's
> in-process proxy. `sf` authenticates with a **placeholder**; the proxy strips it
> and injects the real session **server-side** for requests bound to the org
> (`Authorization: Bearer` for REST, `X-SFDC-Session` for the Bulk API). So the
> real bearer token **never reaches the subprocess**, is **never** written to
> `GITHUB_ENV` or exposed as an input/output, and is **revoked server-side** when
> the action finishes — preserving the mint→use→revoke model even for the managed
> CLI.

## What it does

1. Takes a declarative action (`SfAction` JSON) describing *what* to do.
2. Installs a version-pinned `@salesforce/cli` into a controlled prefix and
  captures the exact `sf` binary path to use.
3. Resolves the action's **minimal capability manifest** — exactly which
  filesystem paths, network hosts, and credentials the `sf` invocation needs,
  derived from the action's inherent risk level.
4. Spawns that managed `sf` confined to the manifest. On Linux runners the
  spawned process is irreversibly restricted via Landlock *before* `exec` (a
  `pre_exec` hook in the forked child); the parent runner stays unconfined.
5. Parses the structured result and persists a **tamper-evident audit record**
   (Blake3 manifest hash, granted capabilities, outcome) as a workflow artifact.

The sandbox is built on the [`nono`](https://docs.rs/nono) capability runtime.
Confinement is **unconditional**: egress is kernel-forced `ProxyOnly` via Landlock
NetPort, which needs **Landlock v4 (Linux kernel 6.7+)**. On a host without that
support, session setup **fails closed** rather than running unconfined.

> **Prerequisite:** the job must have Node/npm available so the action can
> install `@salesforce/cli` into a controlled prefix. GitHub-hosted runners do.
> Authentication is handled in-process via OIDC — set `target-instance` and grant
> `permissions: id-token: write` (see [Authentication](#authentication-zero-stored-credentials));
> you do **not** need a separate auth/login step.

## Inputs

| Input | Default | Description |
|---|---|---|
| `action` (required) | — | The declarative action as `SfAction` JSON. |
| `target-instance` | `` | Busbar-equipped org instance URL to mint a session against via GitHub OIDC. **Required for the default auth path** (with `permissions: id-token: write`). |
| `eca-client-id` | `` | Consumer key of the `BusbarGitHubEca` in the Busbar-equipped org (non-secret). Defaults to the baked PBO-pinned public key. |
| `token-handler` | `` | Apex token-exchange handler dev name. Defaults to `GitHubTokenExchangeHandler`. |
| `oidc-audience` | `` | OIDC token audience. Defaults to `target-instance`. |
| `sf-access-token` | `` | **Advanced / local-dev override only.** Pre-obtained token; skips OIDC. Leave empty in CI. |
| `sf-instance-url` | `` | **Advanced / local-dev override only.** Instance URL paired with `sf-access-token`. Takes precedence over `target-instance` when set. |
| `sf-cli-version` | `latest` | npm version spec for `@salesforce/cli` to install. Pin an exact version if you want reproducibility. |
| `working-dir` | `` | Working directory for the `sf` invocation. |
| `output` | `` | Path to write the structured `sf` result JSON. |
| `audit-out` | `.busbar/sandbox-audit.json` | Path to write the JSON audit log. |
| `version` | `latest` | `sf-cli` binary release tag. |
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
| `sf-cli-path` | Path to the managed `sf` binary installed for this run. |
| `sf-cli-prefix` | Prefix directory containing the managed `sf` install. |
| `sf-cli-version` | Version of `@salesforce/cli` that was installed. |

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
[`SfAction`](../../busbar-extensions/crates/sf-cli/src/action.rs)
for the full vocabulary.

## Example

The default, recommended path: OIDC self-mint. Note `permissions: id-token: write`
(required so GitHub injects the OIDC endpoint) and `target-instance` (the
Busbar-equipped org). The `target_org` in the action JSON is the alias the binary
registers the minted session under.

```yaml
permissions:
  contents: read
  id-token: write                 # REQUIRED: lets sf-cli mint a session via OIDC

jobs:
  query:
    runs-on: ubuntu-latest        # Linux → Landlock enforcement
    environment: devhub-pbo-scratch
    steps:
      - uses: busbar-actions/sf-cli@v1
        id: q
        with:
          target-instance: ${{ vars.SF_INSTANCE_URL }}   # Busbar-equipped org instance URL
          sf-cli-version: 2.69.6
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

No `org-auth`/login step is needed — the binary mints, uses, and revokes the
session itself. There is no token in `GITHUB_ENV` and none passed between steps.

## Platform requirements

Confinement is kernel-enforced and unconditional — there is no unconfined mode.
The `ProxyOnly` egress boundary requires **Landlock v4 (Linux kernel 6.7+)**; on a
host without it, session setup **fails closed** rather than running unconfined. The
published composite action currently ships a Linux x86_64 binary only, and
GitHub-hosted `ubuntu-latest` runners meet the kernel requirement.
