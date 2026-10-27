# Making cli-top easy for coding agents

Research date: 2026-10-27

Method: current primary documentation plus static inspection of this checkout.
No live VTOP authentication, portal submissions, latency benchmarks, or agent
success-rate measurements were performed. Proposed commands below are designs,
not features already implemented.

## Executive assessment

`cli-top` has a stronger machine-facing core than its public help suggests. The
hidden `proxy` command already returns one JSON object with status, timing,
structured table data, and typed errors. `CLI_TOP_PROXY_MODE=1` also suppresses
terminal chrome and turns interactive selectors into explicit
`selection_required` requests. These are the right primitives for an agent.

The weakness is discoverability and consistency. A coding agent must know about
an undocumented `proxy` entry point, supply credentials as positional
arguments, translate flags into a second protocol, and handle ordinary commands
whose output is primarily terminal text. Public commands do not expose a common
`--json`, `--non-interactive`, or `--select` contract, and startup can still do
network/telemetry work before the command runs.

## What the reference tools do well

### Vercel CLI

Vercel documents both human use and automation as first-class use cases. Its CI
guidance prefers `VERCEL_TOKEN` over a token in command-line arguments, because
arguments can appear in process listings and logs. Its newer integration flow
adds task-shaped commands (`discover`, `guide`, `add`) and a documented
`--format=json` output mode, so an agent can discover capabilities and consume
stable data without scraping prose.

Useful patterns for cli-top:

- environment-based secrets with explicit precedence;
- a documented machine output flag on normal commands;
- task-level discovery and setup guidance;
- stable, versioned output schemas.

The integration resource commands provide an especially useful concrete
pattern: when confirmation is missing in non-interactive use, they return a
structured `confirmation_required` error, a `next` array of suggested commands,
and exit code 1. JSON output does not itself grant consent to mutate resources.
This is documented for those commands; it is not a claim that every Vercel
command has identical output. [Vercel integration reference](https://vercel.com/docs/cli/integration).

### Cloudflare `cf`

Cloudflare presents one CLI across API resources and Workers projects. Its docs
explicitly say coding agents run the same commands, provide typed
`cloudflare.config.ts` project configuration, and expose `cf cli search` for
plain-language command discovery. Most of its large command surface is
generated from API schemas, which keeps names, flags, and validation coherent.

Useful patterns for cli-top:

- make the agent path the normal CLI path rather than a private bridge;
- give agents a command search/catalog endpoint;
- generate or validate schemas from the same types used by the implementation;
- keep project-local configuration typed and inspectable.

Its agent workflow is more specific than plain help: search locally without
credentials (`cf cli search`), inspect a generated API request (`cf schema`),
then preview with `--dry-run`. Schema coverage is limited to generated API
commands; other commands still use `--help`, and some request-body schemas are
incomplete. `cf` remains beta, so its interface is not a stability benchmark.
[Cloudflare agent documentation](https://developers.cloudflare.com/cf/agents/).

### Vercel `agent-browser`

`agent-browser` is designed around a low-token observation/action loop. A
compact accessibility snapshot assigns stable `@eN` references; the agent acts
on those references instead of reconstructing selectors from full HTML. JSON
responses expose structured refs, and `--delta` returns bounded changes after a
baseline. The tool also has semantic locators, explicit waits, scoped
snapshots, named tabs, and failure-before-action checks.

Useful patterns for cli-top:

- return compact structured observations, not only pretty tables;
- give every selectable row a stable ID and accept that ID on the next call;
- return deltas or resumable state for long workflows;
- distinguish “action did not run” from “action ran and failed”;
- make waits, scope, and context explicit.

The lesson is to return actionable identifiers alongside observations. Browser
refs are scoped to browser state and have explicit invalidation rules; cli-top
should prefer VTOP resource IDs, not copy ephemeral browser references.

### GitHub CLI (`gh`)

`gh` adds another useful pattern: commands with JSON support allow field
selection and in-process `--jq` filtering. Its environment contract separately
controls prompts, update notices, telemetry, authentication, and repository
context. cli-top could similarly offer `--fields`/`--course-id`/`--date` filters
and explicit account/config selection. This cuts context and ambiguity before
the agent reads the result. [Formatting](https://cli.github.com/manual/gh_help_formatting),
[environment](https://cli.github.com/manual/gh_help_environment).

## Findings in this checkout

| Evidence | Consequence for an agent | Recommended change |
| --- | --- | --- |
| [`cmd/proxy_command.go`](../cmd/proxy_command.go): hidden Cobra command, credentials in argv, own dispatch switch | Public help does not expose the machine path; secrets travel in shell arguments | Public output mode using the existing authenticated session; compatibility adapter for proxy |
| [`helpers/ux.go`](../helpers/ux.go): `CommandRunner` always prints a success footer after a void callback | Printed feature failures can be followed by a green success message | Return errors through the execution layer |
| [`features/marks.go`](../features/marks.go): `GetMarks` returns no error; selection failures print then return; fetch errors are only printed in debug | A proxy dispatch can return `nil` despite a failed fetch; no-data and failure can be conflated | Return typed data plus errors; keep empty success distinct from failed retrieval |
| [`cmd/proxy_command.go`](../cmd/proxy_command.go): proxy error branches print and return from `Run`, rather than `RunE` | JSON can say `success:false` while the process exits zero | Match process exit status to the result |
| [`internal/proxyutil/proxyutil.go`](../internal/proxyutil/proxyutil.go): flags are manually parsed; `--semester=1` is treated as a different flag name; unknown flags are retained | Familiar Cobra syntax behaves differently on the proxy path; invalid input can silently fall back | One validated parser and registry; reject unknown flags before login |
| [`helpers/table-renderer.go`](../helpers/table-renderer.go): plain selectors ignore `ReadString` errors in a retry loop outside proxy mode | Closed stdin can cause repeated prompts instead of a bounded failure | Detect non-TTY input and EOF; emit selection-required and terminate |
| [`helpers/sem-details.go`](../helpers/sem-details.go): proxy mode without a semester selects the last element | The chosen semester is implicit and has mode-specific semantics | Accept stable semester IDs and return the resolved semester and default rule |
| [`helpers/table-renderer.go`](../helpers/table-renderer.go) and [`internal/proxyutil/proxyutil.go`](../internal/proxyutil/proxyutil.go): snapshots contain string rows, while prose is included in both `output` and `messages` | JSON still requires table/prose interpretation and can duplicate tokens | Typed domain objects, IDs, numbers, nulls, and ISO dates; optional human text |
| [`cmd/start.go`](../cmd/start.go): startup runs before Cobra validation; decommission/browser branches print and return | Invalid commands can do network work; early exits can violate machine output expectations | Validate locally first; encode required startup policy failures in the same result contract |
| [`features/events.go`](../features/events.go): proxy mode lists events and returns, while human mode offers registration | An action accessible to humans has no equivalent public non-interactive path | Explicit list/register verbs with stable event IDs |

Existing strengths worth preserving: side-effect-free help/version paths,
semester caching, bounded session refresh, facility `--confirm`, leave
`--apply` validation, and proxy `sync` with per-command partial results. Avoid
reimplementing these features merely to rename the CLI.

## cli-top compared with these patterns

| Agent concern | cli-top today | Reference pattern | Priority |
| --- | --- | --- | --- |
| Discoverability | Cobra help; hidden `proxy`; no command search | `cf cli search`, Vercel task-shaped commands | P0 |
| Machine output | JSON only through `proxy`; normal commands print text | Vercel `--format=json`; agent-browser `--json` | P0 |
| Authentication | `login --username/--password`; proxy positional credentials | Vercel env token recommendation | P0 |
| Interaction | Numeric/fuzzy prompts; proxy emits selection requests | Stable refs and semantic actions | P0 |
| Error contract | Proxy has `auth_failed`, `selection_required`, `execution_error`; public commands mix prints and errors | Typed, action-safe errors | P0 |
| State | Session/config and semester cache exist; selection state is request-local | Durable refs, named tabs, deltas | P1 |
| Output efficiency | Tables are captured, but text is also captured and often verbose | Compact snapshots/deltas | P1 |
| Side effects | Help/version avoid startup effects, but ordinary startup performs kill-switch, UUID registration, update checks | Explicit CI/automation modes | P1 |
| Schema/tooling | Internal `TableSnapshot` and proxy schema; no published schema or skill | Typed config/generated API schemas | P1 |

## Recommended product contract

Make these flags persistent on every read command:

```text
cli-top attendance --semester 3 --output json
cli-top attendance --semester 3 --output table   # default in a TTY
cli-top attendance --semester 3 --non-interactive
```

`--output json` should write exactly one JSON document to stdout and diagnostics
to stderr. Errors belong in the same JSON envelope; stderr is for optional
human diagnostics. The envelope should be stable and small:

```json
{
  "schema_version": "1",
  "success": true,
  "command": "attendance",
  "data": { "semester_id": "<portal-id>", "courses": [] },
  "warnings": [],
  "request_id": "...",
  "duration_ms": 123
}
```

Errors should use stable `kind`, `message`, and optional machine fields such as
`retryable`, `required_flags`, and `choices`. Exit codes should distinguish
usage, authentication, selection-required, upstream, and partial-success
failures. Never mix banners, spinners, or success lines into JSON stdout.

For selectors, expose stable option IDs and accept them directly:

```json
{
  "kind": "selection_required",
  "selection": {
    "id": "semester",
    "options": [{"id":"sem-3","label":"Fall 2026","index":3}]
  }
}
```

An agent can then call `--semester-id sem-3` (or the equivalent command flag)
without depending on display order. Keep numeric and fuzzy input for humans.
The `sem-3` value is illustrative; use the actual stable portal ID. A default
semester must be reported in the result with its selection rule.

Domain fields should carry raw values: `attended: 28`, `held: 35`,
`percentage: 80`, rather than a cell such as `28/35 (80%)`. Schedule dates
should carry `Asia/Kolkata` or an explicit UTC offset. Include `fetched_at` and
cache/source metadata when relevant, so an agent can assess freshness. Download
results should include absolute paths, byte counts, content types, and
per-file failure status; opening a folder should require an explicit flag.

Separate reads and writes with explicit commands or flags. For example,
`events list`, `events register --event-id ...`, `leave status`, and
`leave apply` can coexist with the current aliases. `--non-interactive` means
never prompt; it must never imply `--confirm`. A dry run must validate and
describe intended changes without submitting a VTOP form. After an ambiguous
submission timeout, return an unknown-outcome error and a status command to
reconcile; do not automatically repeat the submission. Use a portal-supported
idempotency key if available, without promising exactly-once behavior when the
upstream cannot provide it.

## Prioritized implementation plan

### P0: make the existing protocol public and safe

1. Add a root `--output table|json` flag and `--non-interactive` flag.
2. Route all command results through a shared result envelope; reuse the
   existing `TableSnapshot` and proxy error types instead of scraping output.
3. Move credentials to `CLI_TOP_USERNAME`/`CLI_TOP_PASSWORD` for automation,
   support stdin or an OS credential store, and mark password flags as unsafe in
   help. Do not require credentials in argv for `proxy`.
4. Publish `cli-top schema` (or `cli-top agent-manifest`) that emits commands,
   flags, required auth, side effects, output schemas, and exit codes as JSON.
5. Ensure JSON mode suppresses animations and browser/file opening and never
   prompts. Give telemetry and update work separate explicit controls. Preserve
   required kill-switch/CAPTCHA policy, but return structured failures such as
   `client_decommissioned` or `human_action_required` rather than opening a
   browser or waiting on stdin. Validate flags and discover schemas locally
   before doing authentication/network work.
6. Fix feature error propagation and process exit status as part of this first
   milestone. JSON syntax alone is insufficient if success is inaccurate.

### P1: improve agent interaction quality

1. Give every selector option a stable ID and add `--select <id>`/semantic flag
   forms to course, faculty, facility, semester, assignment, and event flows.
2. Add structured `next` actions and include scoped choices in selection errors.
   Stateless reruns with stable IDs should come first; add resumable tokens only
   for workflows that demonstrably need them.
3. Add server-side/domain filters, field selection, and bounded lists before
   considering delta responses. A semester timetable usually needs a single
   compact read; browser-style deltas add unnecessary state until repeated
   polling is a real workload.
4. Add `cli-top search <natural-language task>` backed by the command manifest;
   at minimum, support local fuzzy search over command names, descriptions, and
   required flags.
5. Document explicit read/write behavior, dry runs, and how to reconcile an
   uncertain submission result. Keep confirmation flags independent of output
   and prompt flags.

### P2: make the contract maintainable

1. Version the JSON schema and publish examples for each command.
2. Generate Cobra flags and manifest metadata from one typed command registry.
3. Add contract tests that assert stdout is parseable JSON, stderr is separate,
   prompts never occur in non-interactive mode, and every error has a stable
   kind and exit code.
4. Ship an agent skill/integration document with copy-paste workflows for
   login, read commands, selection continuation, downloads, and retries.

Use fixture-based agent tasks to judge whether the work helped: list semesters
and fetch marks for a specified semester; identify tomorrow's classes; recover
from ambiguous course input; download one material; handle a rate limit; and
reconcile a timed-out leave application without submitting twice. Measure task
completion, tool calls, output tokens, incorrect selections, and unintended
mutations. No numerical improvement is claimed without this evaluation.

## Suggested first milestone

Implement the shared JSON envelope, `--output json`, `--non-interactive`, safe
environment-based authentication, and `cli-top schema`. Keep `proxy` as a
backward-compatible adapter over that same execution layer. This removes the
largest integration tax. Existing table capture and selection hooks can support
a transitional compatibility payload, but reliable error propagation requires
changes inside feature functions. Prioritize typed results for attendance,
marks, timetable, and planning before converting the entire catalog. A single
execution/result layer should feed terminal formatting, JSON, proxy, and MCP.

## Sources

- [Vercel CLI overview](https://vercel.com/docs/cli)
- [Vercel CLI agent-oriented integration output](https://vercel.com/changelog/vercel-cli-for-marketplace-integrations-optimized-for-agents)
- [Cloudflare CLI](https://developers.cloudflare.com/cf/)
- [Cloudflare CLI for agents](https://developers.cloudflare.com/cf/agents/)
- [Cloudflare CI contract](https://developers.cloudflare.com/cf/ci/)
- [Vercel integration reference](https://vercel.com/docs/cli/integration)
- [GitHub CLI output formatting](https://cli.github.com/manual/gh_help_formatting)
- [GitHub CLI environment contract](https://cli.github.com/manual/gh_help_environment)
- [agent-browser snapshot and refs](https://github.com/vercel-labs/agent-browser/blob/main/skill-data/core/references/snapshot-refs.md)
- [agent-browser command reference](https://github.com/vercel-labs/agent-browser/blob/main/skill-data/core/references/commands.md)
