> [!WARNING]
> **`busbar-actions` is under heavy active development — expect breaking changes.**
> These repositories are public, but **not ready for use yet** — please don't depend on them.
> A pilot is starting soon: **[star and watch the busbar-actions organization](https://github.com/busbar-actions)** for the launch of Discussions and the pilot announcement.

# busbar-actions/sf-cli

Run a single **declarative Salesforce CLI action** inside a capability-based
sandbox — kernel-enforced confinement (Linux [Landlock](https://docs.kernel.org/userspace-api/landlock.html))
on GitHub-hosted runners — with a full audit trail.

The action first installs `@salesforce/cli` (the version you name, `latest` by
default) into a controlled prefix, then executes the declarative action against
that managed binary. You do not need to preinstall `sf` on `PATH`, and you do
not need to pin a CLI release: every request `sf` sends is decided by what it
does, not by which release sent it.

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

1. Grant `permissions: id-token: write` (so GitHub injects the OIDC endpoint)
   and `contents: read` (so the supervisor can read your repository's
   policies with `github-token`, by default the job's own token).
2. Set `target-instance` to the Busbar-equipped org's instance URL.

> ### How the managed CLI stays token-free
>
> Every other Busbar action binary keeps the minted token **only in zeroizing
> memory** and calls the Salesforce API directly. `sf-cli` is different: it shells
> out to the managed `@salesforce/cli`, which persists auth to its HOME. Rather
> than hand `sf` the real token, the run is confined to a **busbar-sandbox**
> session whose network egress is kernel-forced (`ProxyOnly`) through the
> supervisor's in-process proxy. `sf` authenticates with a **placeholder**. The
> proxy shows the Salesforce request policy **every request in full** — request
> line, headers, body — and injects the real session **server-side** only into a
> request the policy allowed (`Authorization: Bearer` and `X-SFDC-Session`). A
> denied request reaches `sf` as **HTTP 403** without the credential. So the real
> bearer token **never reaches the subprocess**, is **never** written to
> `GITHUB_ENV` or exposed as an input/output, and is **revoked server-side** when
> the action finishes — preserving the mint→use→revoke model even for the managed
> CLI.
>
> **If the request policy cannot inspect a request, the request gets no
> credential.** There is no unenforced mode, opt-out or toggle, on any org kind.
>
> The SOQL a run may send is parsed only in a **confined, credential-free
> helper**: a copy of the `sf-cli` binary with no network, an empty
> environment and nothing readable but itself, started before any credential
> exists. The supervisor checks every answer against the bytes that were sent
> and makes every decision itself.

## What it does

1. Takes a declarative action (`SfAction` JSON) describing *what* to do.
2. Installs `@salesforce/cli` at the version you name into a controlled prefix
  and captures the exact `sf` binary path to use.
3. Refuses, **before a session is minted**, any action whose requests the policy
  cannot inspect in this release (see [Variant availability](#variant-availability)),
  a `data_query` whose SOQL does not parse, and a `working-dir` that holds a Git
  directory.
4. Mints the session, then — before the proxy or `sf` exists — reads your
  repository's **policies** (`.busbar/policies/*.cedar` at the head of the
  default branch; see [Repository policies](#repository-policies)) and its
  **door configuration** (`.busbar/door.toml` at the same commit, if there is
  one; see [Header forwarding](#header-forwarding-busbardoortoml)), resolves a
  `data_query` against the org's describes, scores it, and writes the run's
  **declaration** to the job log. A policy that forbids what the run declared,
  or an invalid policy set or door configuration, stops the run here.
5. Spawns that managed `sf` confined: read-only `/`, a private `HOME` and
  `TMPDIR` created for the run (and `working-dir`, if given) as the only writable
  paths, an explicit environment (nothing is inherited from the job; no
  `ACTIONS_*`, `GITHUB_*`, `RUNNER_*`, `LD_PRELOAD`, `LD_AUDIT` or `DYLD_*`
  variable is ever set), and TCP to the proxy's port only. On Linux runners the spawned process is irreversibly
  restricted via Landlock plus a seccomp network baseline (UDP, raw sockets and
  `AF_UNIX` stream sockets denied) *before* `exec`; the parent runner stays
  unconfined.
6. Parses the structured result and persists a **tamper-evident audit record**
   (Blake3 manifest hash, granted capabilities, outcome, and the request
   policy's report with the whole **run context**) as a workflow artifact.

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
| `sf-cli-version` | `latest` | npm version spec for `@salesforce/cli` to install. Any release works: the CLI's own org-resolution reads are admitted by what they read (read-only queries of `Organization`, `ScratchOrgInfo`, `SandboxProcess` and the run-as user's own `User` record, no credential-bearing field, any API version), not by their wording. Pin a version only for reproducibility. |
| `github-token` | `${{ github.token }}` | Token the supervisor uses to read your default branch, `.busbar/policies` and `.busbar/door.toml` through the GitHub REST API; needs `contents: read`. Never given to `sf`. |
| `working-dir` | `` | Working directory for the `sf` invocation (writable by `sf`). Refused before a session is minted when it is, or contains, `/tmp`, `RUNNER_TEMP`, a `GITHUB_ENV`/`GITHUB_OUTPUT`/`GITHUB_PATH`/`GITHUB_STATE`/`GITHUB_STEP_SUMMARY` file or the runner's `HOME`; when it is inside a Git directory or contains a `.git` entry at any depth (a directory, a worktree or submodule `gitdir:` file, or a symbolic link), because `sf` could otherwise plant `core.fsmonitor` or `core.hooksPath` for the job's later `git` commands; or when any directory in it cannot be read. Use a directory that holds no checkout. |
| `output` | `` | Path to write the structured `sf` result JSON. |
| `audit-out` | `.busbar/logs/sandbox-audit.json` | Path to write the JSON audit log (each entry carries the request policy's redacted report under `.policy`). |
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
{ "action": "data_query", "target_org": "my-org", "soql": "SELECT Id, Name FROM ApexClass", "tooling": true }
```
```json
{ "action": "org_display", "target_org": "my-org" }
```
```json
{ "action": "org_list" }
```

See the crate's
[`SfAction`](../../busbar-extensions/crates/sf-cli/src/action.rs)
for the full vocabulary.

## Variant availability

Every request `sf` sends is decided by the Salesforce request policy on the
full request before the credential is attached. An action whose requests carry
a body the policy cannot yet decode has **no grant** in this release: `sf-cli`
refuses it before minting a session (a `policy_gate` failure, exit 1), on every
org kind. No input, environment variable or org kind enables it.

| Action | Available | Refusal |
|---|---|---|
| `data_query` | yes | — (a declared query; see [The `data_query` SOQL](#the-data_query-soql)). |
| `org_display` | yes | — |
| `org_list` | yes | — |
| `data_export` | no | `sf data export bulk` sends a Bulk API 2.0 query-job body (POST …/jobs/query) that this release cannot inspect. Use `data_query`, or `busbar-actions/cci-run` `task_query`. |
| `apex_run_tests` | no | `sf apex run test` sends a Tooling runTestsAsynchronous JSON body that this release cannot inspect. |
| `project_deploy` | no | `sf project deploy start` sends Metadata API SOAP (or REST multipart) bodies, with the session inside the envelope, that this release cannot inspect. |
| `deploy_status` | no | `sf project deploy report` sends Metadata API SOAP (or REST multipart) bodies that this release cannot inspect. |
| `project_retrieve` | no | `sf project retrieve start` sends Metadata API SOAP retrieve and checkRetrieveStatus bodies that this release cannot inspect. |
| `apex_execute` | no | `sf apex run` sends an Apex SOAP executeAnonymous body that this release cannot inspect. Use `busbar-actions/cci-run` `task_execute_anon`: Apex committed in your repository, run only when your repository's policies permit it. |
| `org_create_scratch` | no | Acts on a DevHub and writes ScratchOrgInfo/ActiveScratchOrg records whose bodies this release cannot inspect. Use `busbar-actions/sf-org-create`. |
| `org_delete_scratch` | no | Same; use `busbar-actions/sf-org-delete`. |

### The `data_query` SOQL

`soql` is the one query `sf` may send, **byte for byte**:

- **Before a session is minted** it is parsed with tree-sitter-sfapex's SOQL
  grammar. Accepted: fields, parent paths, polymorphic paths, `TYPEOF`, one
  level of child-relationship subqueries in the select list, aggregates
  (`COUNT`, `COUNT_DISTINCT`, `SUM`, `MIN`, `MAX`, `AVG`), `GROUP BY` /
  `ROLLUP` / `CUBE`, `HAVING`, functions (`toLabel`, `convertCurrency`,
  `FORMAT`, date functions, `DISTANCE`/`GEOLOCATION`), `FIELDS(STANDARD |
  CUSTOM | ALL)`, `WHERE` (including semi-joins and anti-joins), `WITH
  SECURITY_ENFORCED | USER_MODE | DATA CATEGORY`, `USING SCOPE`, `ORDER BY`,
  `LIMIT`, `OFFSET` and `FOR VIEW | REFERENCE | UPDATE`. Refused: anything
  that does not parse, `WITH SYSTEM_MODE`, `ALL ROWS`, `UPDATE TRACKING` /
  `VIEWSTAT`, unknown functions, nested subqueries, bind variables, comments,
  a name spelled like a SOQL keyword, and `AND`/`OR`/`NOT` mixed without
  parentheses (write `A OR (NOT B)`). A refusal names the reason and a byte
  offset, never the query text.
- **When the session is bound**, it is parsed again in the credential-free
  helper, and every object and field **any clause** references — the select
  list, parent paths, polymorphic targets, `TYPEOF`, semi-joins, child
  subqueries, function arguments, `GROUP BY`, `HAVING`, `ORDER BY`, and
  `FIELDS()` expanded through the describe — is resolved against the org's
  describes (the Tooling API's when `"tooling": true`) and checked against
  the credential-bearing objects and fields by its resolved API name. A name
  that does not resolve, or resolves to a credential-bearing object or field
  (for example `(SELECT … FROM OAuthTokens)`, or `FIELDS(ALL)` on an object
  with a credential-bearing field), refuses the run before `sf` starts. The
  query is then scored (see [Risk scores](#the-run-context-and-risk-scores))
  and your repository's policies are asked about it.
- **At the proxy**, only that query is admitted, plus `sf`'s `columns=true`
  request for the same query and its next pages (at the query locator
  Salesforce returned for it), and the CLI's own org resolution **by what it
  reads**: read-only queries whose objects are all org-identity objects
  (`Organization`, `ScratchOrgInfo`, `SandboxProcess`, and the run-as user's
  own `User` record) with no credential-bearing field, at any API version,
  the org's own record, `userinfo` and API discovery. Nothing else is
  readable. Before the policy sees a request, the proxy removes every header
  whose name it does not forward, and the request goes on without it; the
  headers it forwards are checked by meaning (content negotiation, content
  type, user agent, the credential headers the proxy replaces,
  `Sforce-Call-Options: client=`, and any your repository lists in
  `.busbar/door.toml`), so a new CLI release that sends the same kinds of
  requests, or a new header, needs no Busbar release.
- The query has to fit the proxy's request line: `sf` sends it URL-encoded
  (spaces, commas and quotes take three bytes each), and a query too long for
  the proxy to read is refused before a session is minted. The proxy reads
  up to Salesforce's own REST URI limit, 16,384 bytes.
- `sf` is pinned to API version 66.0 (`SF_ORG_API_VERSION`), the version the
  query is described at; the query, its column request and its pages are
  admitted at that version only. Only the head of the first page is read by
  the policy; the rest of every page streams through unbuffered.
- Known limits, all of which fail the run rather than read anything
  unchecked: pages of a child subquery's rows (more child rows than
  Salesforce returns per parent) are denied.

## Repository policies

Every run reads your repository's **`.busbar/policies/*.cedar`** from the head
of its **default branch** when the run starts, validates them strictly
against the Busbar Cedar schema (`busbar-cedar/1`), and applies them after
Busbar's compiled-in Baseline and the action's grant: a `forbid` can stop any
request (a `permit` never allows anything the Baseline or the grant denies).
With no policy files, nothing changes for `sf-cli`'s actions. An invalid
policy set fails the run before `sf` starts. Policies are asked once before
`sf` starts, about the declared query, and again about every request.

```cedar
// Never read contact data (email, phone, address fields) from production.
@id("no-contact-data-in-production")
forbid (principal, action == Busbar::Action::"query", resource)
when {
  resource.org_kind == "production" &&
  context.query.sensitive_classes.contains("contact_info")
};
```

```cedar
// No query Busbar scores 70 or more against production.
@id("no-high-risk-soql-in-production")
forbid (principal, action, resource)
when {
  resource.org_kind == "production" &&
  context.query.present && context.query.risk >= 70
};
```

```cedar
// Narrow the CLI's own org resolution: no sandbox lookups from production.
@id("no-sandbox-lookup-in-production")
forbid (principal, action == Busbar::Action::"query", resource)
when {
  resource.org_kind == "production" &&
  context.query.org_resolution &&
  context.query.objects.contains("sandboxprocess")
};
```

`org_display` and `org_list` let the CLI read what it asks for, checked
lexically, so their queries are not parsed and score **100**: a policy like
the second one above forbids them in production. The full reference — what
`principal`, `action`, `resource` and `context` carry, how policies combine,
more examples — is in
[busbar-actions/cci-run](https://github.com/busbar-actions/cci-run#writing-repository-policies).

### Header forwarding (`.busbar/door.toml`)

The proxy forwards only the headers the policy checks by meaning, and the
ones it must see to refuse a request (method overrides,
`Content-Encoding`, the hop-by-hop names it does not handle itself); it
removes every other header before the policy sees the request, and the
request goes on without it. A header a new `sf` release adds never breaks a
run and never reaches your org. To have another header forwarded, list it
on your default branch:

```toml
# .busbar/door.toml
forward_headers = ["traceparent"]
```

`forward_headers` is the only key (at most 64 names, each listed once). A
credential, framing or hop-by-hop, method-, host- or target-override, proxy
or client-IP header can never be listed, in any spelling (`_` is read as
`-`). An invalid file fails the run before `sf` starts. The declaration records the file (`declaration.door`), and the
audit log counts the header names the proxy removed, never their values
(`stripped_headers`). The full rules are in
[busbar-actions/cci-run](https://github.com/busbar-actions/cci-run#header-forwarding-busbardoortoml).

## The run context and risk scores

Every run keeps a redacted, append-only **run context**: a **declaration**
written before `sf` starts (the action, the declared query by digest,
resolved objects, sensitive field classes, shape and risk, the grant, the
job's identity from its signed OIDC claims, the org, where the policies came
from, the scorer versions), a **journal** of every request the proxy decided
(with its operation, verdict and the policies that decided it), every side
effect and every fact learned, and **derived facts** (maximum and cumulative
risk, queries, objects touched). The declaration is logged in the job log
before `sf` starts; the whole context is in the audit log, under
`.policy.run_context` on each entry. Nothing in it is a token, a record value
or query text.

The declared query's **risk score** (`busbar/soql-risk/v3`, 0–100, with its
reasons) is in the job summary ("Declared query risk"): for example
`row-access` +10, `no-filter` +10, `no-limit` +10, `contact-info` +15,
`personal-identity` +20, `bulk-sensitive-read` +45,
`credential-adjacent-object` +20, `for-update` +15. Nothing in the query's
text lowers it (`WITH SECURITY_ENFORCED` and `WITH USER_MODE` narrow nothing
through the API), and a `WHERE` clause does not bound it: any query that
returns personal data or secrets for more than 200 records (no `LIMIT`, or
a `LIMIT` over 200) scores 70 or more. Object names in
`context.query.objects` and `context.run.objects_touched` are lower-case
(`"sandboxprocess"` above); a policy spelling one with an upper-case letter
does not load. The full table is in
[busbar-actions/cci-run](https://github.com/busbar-actions/cci-run#risk-scores).

### What changed for the sandboxed `sf`

- The workflow's `env:` no longer reaches `sf` (for example
  `SF_ORG_METADATA_REST_DEPLOY`): `sf` gets only an explicit set of variables,
  a private `HOME`/`TMPDIR`, and `SF_DISABLE_DNS_CHECK=true` (the sandbox denies
  UDP, so `sf`'s DNS pre-check would fail) and `SF_USE_GENERIC_UNIX_KEYCHAIN=true`.
- `SF_SANDBOX_ALLOW_HOSTS` is gone: the proxy serves only the org's API host.
- Policy denials reach `sf` as **HTTP 403** (they used to be 503). A request
  line longer than the proxy reads gets **HTTP 414**.
- `data_query` is a declared query (see above); it no longer reads anything
  else in the org. It now accepts aggregates, grouping, functions,
  `FIELDS()`, `WITH`, `USING SCOPE`, `OFFSET` and `FOR`, every field any
  clause names resolved and checked.
- Any `sf` release works: the CLI's own org-resolution reads and headers are
  decided by what they are, not matched against one release's exact bytes.
- A header the proxy does not forward is removed before the policy sees the
  request, and the request goes on without it (it used to be refused). List
  one in `.busbar/door.toml` on your default branch to have it forwarded.
- Every run reads your repository's `.busbar/policies/*.cedar`, so the job
  needs `permissions: contents: read` (and `github-token`, by default the
  job's own token). A run whose policies do not load, or that a policy
  forbids, fails before `sf` starts.
- The declaration is logged before `sf` starts, and the audit log carries
  the run context and the declared query's risk score (also when the
  placeholder login fails).
- A `working-dir` holding a Git directory is refused before minting.
- The job summary and the audit log (`.policy` on each entry) carry the
  redacted policy report: the org kind, every decision (each allow with
  `"inspection": "full"`) and the claim status. The claim status is
  `ineligible` on every run today: the org's trust rule does not yet pin the
  executing workflow, and a sandboxed process can reach the proxy's port on
  any address (GHSA-6hww). `org_display` and `org_list` also report
  `soql_not_parsed`: they may read anything the CLI asks for, checked only
  lexically against credential-bearing names.

## Example

The default, recommended path: OIDC self-mint. Note `permissions: id-token: write`
(required so GitHub injects the OIDC endpoint) and `target-instance` (the
Busbar-equipped org). The `target_org` in the action JSON is the alias the binary
registers the minted session under.

```yaml
permissions:
  contents: read                  # REQUIRED: sf-cli reads .busbar/policies
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
The GitHub token (`github-token`) is read by the supervisor alone, for the
repository's policies.

## Platform requirements

Confinement is kernel-enforced and unconditional — there is no unconfined mode.
The `ProxyOnly` egress boundary requires **Landlock v4 (Linux kernel 6.7+)**; on a
host without it, session setup **fails closed** rather than running unconfined. The
published composite action currently ships a Linux x86_64 binary only, and
GitHub-hosted `ubuntu-latest` runners meet the kernel requirement.
