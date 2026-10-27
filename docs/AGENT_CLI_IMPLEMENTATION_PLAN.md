# Implementation plan: a public agent interface for cli-top

Scope: (1) typed errors and exit statuses, (2) public JSON/non-interactive modes,
(3) typed domain results and stable selectors, and (4) scoped schemas and
structured recovery. This document records the original design. The implemented interface and migration notes are in [AGENT_INTERFACE.md](AGENT_INTERFACE.md); implementation details may differ from this baseline.

## 1. What the checkout requires

The change crosses the execution path, not just the renderer:

| Area | Current implementation | Consequence |
| --- | --- | --- |
| Entry point | `main.go` calls `cmd.Execute()`; `cmd/start.go` does startup work before Cobra execution and calls `os.Exit(1)` | Flag errors can trigger network work; early policy returns exit successfully; hard to exercise failures in process |
| Command setup | Global Cobra commands and shared flag variables in `cmd/start.go`; `cmd/creds.go:init` reads config and registers login | Repeated execution shares state; constructing help/schema can resolve/create configuration directories |
| Feature errors | Most feature entry points return nothing; `CommandRunner` unconditionally emits success; some errors only affect flow under `debug.Debug` | A wrapper cannot reconstruct truthful success from captured text |
| Auth | `login.Login` returns cookies, `HomePage` returns cookies/reg-no, invalid states become empty values; `ValidateLogin` prints | Network failure, rejected credentials, and parser failure can become indistinguishable |
| HTTP | `FetchReq` returns ordinary errors and performs bounded session refresh; `FetchReqClient` is a separate path | Preserve cause/status at these boundaries rather than match prose later |
| Output | Direct `fmt` calls throughout features; table hook and proxy capture replace process-global stdout | New JSON should serialize returned values, not use process-global capture |
| Interaction | `CLI_TOP_PROXY_MODE` controls presentation, prompts, defaults, and opening files; several readers ignore EOF | Need independent execution policy and typed selection results |
| Proxy | Hidden command, separate flag parser/dispatch; errors are printed from `Run`; `sync` aggregates per-command results | Keep its wire adapter while replacing the duplicate execution path |
| Data | Existing structs include useful portal IDs, but many parse paths discard identity or return display strings | Retain raw values before generating terminal text |
| Tests | Good mocked transport coverage; executable smoke helper assumes all commands succeed | Reuse fixtures and add assertions for error exits, data, and side effects |

Particular dependencies discovered during inspection:

- Attendance already has `fetchAttendanceRecordsForSemester`,
  `ExtractAttendanceRecords`, and numeric attendance calculations. It discards
  course code and joins counts into a display string before returning records.
- Marks has reusable `subjectDetails`, but `ExtractMarks` returns string rows.
  `GetMarks` can continue parsing after a fetch error. Grades similarly mixes
  extraction and printing in `findAndSaveGrade`.
- Exams has typed `types.ExamEvent` and a fetch/parse split, but the top-level
  command also generates and uploads ICS files. Timetable suppresses its export
  in proxy mode; the exam exporter does not have the same top-level guard.
- Planning uses timetable, calendar, attendance, and exams. `fetchCalendarEntries`
  skips failed months; `getClassGroups`/`getDateList` return data without errors.
  These signatures must change to make partial results truthful.
- Semester rules differ: attendance/exams search candidates for data; marks and
  grades use `SelectSemester`; timetable has a date-dependent choice;
  holiday/planners use the latest entry; consolidated course pages derive a
  separate semester list from course-label text.
- The timetable parser keys courses by subject name. Planner attendance matching
  uses normalized names. A DTO wrapper alone cannot recover missing identity.
- Consolidated/archive course IDs, curriculum course IDs, faculty ERP IDs,
  class IDs, and material IDs are distinct namespaces. A single unqualified
  global `course_id` promise would be misleading.

## 2. Target architecture and proposed files

Use one execution contract. Keep existing parsing/business rules in `features`
and shared transport/session support in `helpers`; avoid a wholesale package
reorganization.

| Proposed file/area | Responsibility |
| --- | --- |
| `internal/contract/errors.go` | Stable kinds, wrapped causes, exit mapping, selection details, structured recovery actions |
| `internal/contract/result.go` | Versioned envelope, warnings, context, partial results |
| `internal/contract/policy.go` | Per-invocation prompt/UI/export policy, independent of output data |
| `types/results.go` and feature-specific result files | Explicit JSON DTOs; no cookies, passwords, CSRF tokens, or credential structs |
| `cmd/root.go`, `cmd/execute.go` | Fresh root command/options per invocation; injectable streams/services; centralized render and exit handling |
| `cmd/registry.go` | Command metadata, handler/result associations, and feature-specific selector policies |
| `cmd/schema.go` | Local command catalog and scoped schema command |
| `cmd/render.go` and feature renderers | JSON encoding and table rendering of the same returned data |
| `helpers/sem-details.go`, `helpers/selection.go` | Fetch lists separately from choosing items; typed choices and ID/index/query resolution |
| `features/*` | Operations return typed results/errors; render adapters preserve human output |
| `cmd/proxy_command.go` | Legacy request/response translation over the same operations |

`internal/contract` must not import `cmd`, `features`, or `helpers`. DTOs must
remain independent of terminal formatting. Dependencies point from commands and
features toward the contract and types, avoiding import cycles.

Give `cmd.Run(args, stdin, stdout, stderr)` an integer return code; `main.go`
alone calls `os.Exit`. Construct a fresh root with invocation-local flag values.
Inject a session provider, transport, clock, and execution policy where used;
do not expand this into a generic framework.

## 3. Shared result and error contract

Proposed public JSON envelope:

```json
{
  "schema_version": "1",
  "command": "attendance",
  "success": true,
  "data": {"courses": []},
  "context": {
    "semester": {"id": "<portal-id>", "name": "<portal-label>"},
    "selection_source": "explicit_id",
    "timezone": "Asia/Kolkata"
  },
  "warnings": [],
  "error": null
}
```

`data` is a documented per-command model. Empty collections are `[]`; absent
numeric/date values are `null`, never zero or a string such as `N/A`. Keep a
normal empty success distinct from a failed fetch. Do not claim `not_published`
unless the portal supplies evidence of that state. Missing expected page
structure is a parse failure, not automatically an empty success.

Proposed exit codes (new contract, not existing behavior):

| Code | Kinds | Policy |
| --- | --- | --- |
| 0 | Successful operation, including valid empty data | `success:true` |
| 1 | `internal_error`, `io_error` | Unexpected/local failure, sanitized message |
| 2 | `invalid_arguments`, `invalid_selection` | No auth/network for syntactically invalid input |
| 3 | `auth_required`, `auth_failed` | Do not disguise transport errors as bad credentials |
| 4 | `selection_required`, `confirmation_required`, `human_action_required` | Return choices/next actions; never wait in non-interactive mode |
| 5 | `network_error`, `upstream_error`, `parse_error` | Keep HTTP status and safe diagnostic context |
| 6 | `client_decommissioned`, `policy_blocked` | Preserve existing server policy; structured failure |
| 7 | `partial_failure` | Useful partial data may accompany `success:false` |
| 130 | `canceled` | Explicit user/process cancellation |

Represent errors with `Kind`, `Message`, `Retryable`, `RequiredFlags`, `Choices`,
`Next`, safe `Details`, and a private wrapped `Cause`. Use `errors.Is/As` and
`Unwrap`, never match error strings. Recovery actions contain canonical command
and argv fields; no raw portal HTML, credentials, cookies, or executable shell
fragments. A suggested write command is advice, not authorization to run it.

Classify HTTP/login failures at their source. Bound retries as today; retryable
describes safe recovery for that operation, not every HTTP POST. VTOP uses POST
for reads as well as writes. This scope adds no broader submission retry logic.

For multi-source operations: missing essential timetable/calendar context fails
the plan; failed attendance/exam enrichment returns explicitly incomplete data
and `partial_failure`, with affected components identified. Validly absent
enrichment is an empty success. Never turn a failed calendar fetch into a
confident holiday/teaching-day assertion.

## 4. Public output and interaction policy

- Add persistent `--output table|json` and `--non-interactive`; keep table output
  the default so existing human commands do not change format unexpectedly.
- JSON mode never prompts and has no terminal chrome. Table mode also never
  prompts when `--non-interactive` is present or required streams are non-TTY.
  Prompt-enabled human mode requires interactive input/output.
- JSON stdout contains one envelope, including feature failures. Optional
  diagnostics go to stderr; debug cannot change results or leak secrets.
- Keep `--help` and completion as plain help/raw shell scripts even with an
  inherited output flag; list these exceptions in schema. Root version can
  return a small JSON result when explicitly requested. Missing/unknown output
  formats are usage failures with a documented diagnostic fallback.
- Use a narrow, side-effect-free bootstrap to recognize output intent even when
  Cobra fails early; respect `--`, flag values, and flag placement. Let Cobra
  validate actual arguments once; bootstrap must not become another parser.
- Validate unknown commands/flags, positional arguments, incompatible flags,
  and leave fields before auth/startup network work. Root/schema construction
  must not load/write configuration; move Viper initialization out of `init`.
- Preserve kill-switch checks on actual portal operations; return structured
  policy errors rather than printing and returning zero or opening a browser.
- Manual CAPTCHA mode returns `human_action_required` without waiting on stdin.
  Do not alter CAPTCHA solving behavior or bypass upstream policy.
- Authenticated reads may maintain their normal local session/cache. JSON reads
  suppress incidental ICS generation/upload and OS open actions. Explicit
  download commands still download the requested files. Existing human export
  behavior can remain behind its renderer/execution policy.
- Keep update/telemetry configuration separate from output selection. Discovery
  does no network work. Portal JSON paths suppress update notices and ensure
  background diagnostics cannot contaminate stdout.

Audit every prompt/open call site: login, update, CAPTCHA, table selectors,
course allocation, material selection, events, facility, leave, nightslip,
`OpenFolder`, `openFile`, and `OpenURLInBrowser`. A mute flag is not proof a
path cannot block. EOF must terminate cleanly in every mode.

## 5. Typed results and stable selection

Separate each feature into fetch/parse, operation assembly, and human rendering.
Preserve numeric attendance thresholds, lab handling, working Saturdays,
holiday rules, timetable semester rules, and current human labels in this work.

First DTOs:

- `SemesterResult`: options with portal IDs/names and cache provenance.
- `AttendanceResult`: code/type, optional portal course identity, faculty,
  attended/held counts, nullable percentage, numeric skip/attend advice and its
  units. Preserve raw class counts separately from lab-session calculations.
- `MarksResult`: course context with typed assessment scores, maxima, weights,
  status, totals, and nullable unavailable values.
- `GradesResult`/`CGPAResult`: numeric credits/scores/CGPA, grade labels/counts.
- `TimetableResult`/`PlanningResult`: deterministic ordered classes, explicit
  dates/timezone/day-order, working Saturdays, exams, calendar status, structured
  advice, and completeness. Replace name-keyed maps where they merge courses.
- `ExamResult`, `CalendarResult`, `HolidayResult`: reuse the existing typed
  dates/events, removing styling from values.

IDs are opaque strings, not parsed integers. Retain portal IDs from option
values, hidden attributes, and existing models. Give identities a namespace and
scope (semester/class/faculty/category/source). Where a view exposes only a
course code, report code/type and nullable ID; do not fabricate a portal ID or
assume a course code is unique across labs, sections, and semesters.

Add exact selectors where supported: `--semester-id`, `--class-group-id`,
`--course-id`, `--faculty-id`, `--category-id`, and `--assignment-id`. Retain
current index/query flags. Reject conflicting ID/index/query inputs rather than
silently choose one. An invalid exact ID never falls back to fuzzy matching or
another semester. Resolve dependent IDs only within their parent scope.

Add list commands:

```text
cli-top semesters --output json
cli-top courses --source consolidated --output json
cli-top courses --source archive --semester-id <id> --output json
cli-top faculties --source archive --semester-id <id> --course-id <id> --output json
cli-top class-groups --semester-id <id> --output json
```

Consolidated course pages currently expose semester labels in course options,
not the same semester-ID list as other features. Verify mappings against actual
responses; where no reliable join exists, expose the source label and accept
the exact course ID without inventing a global semester relationship. Similarly,
use ERP/class/material identifiers only in endpoints that actually accept them.
Discovery responses and schema must disclose these scopes.

Default rules are explicit command metadata. Preserve existing deliberate
recent-semester search for attendance/exams, with checked candidates and chosen
semester reported. An explicit semester disables fallback. Distinguish all
retrievals failed from all successful retrievals empty, and disclose failures
encountered during fallback. Human-only pickers become `selection_required` in
machine mode unless the command has a documented deterministic default. Do not
use JSON formatting itself to choose a different semester.

## 6. Scoped schemas and recovery

```text
cli-top schema
cli-top schema attendance
cli-top schema course-page
```

Schema discovery always returns JSON, needs no credentials, and performs no
portal, kill-switch, update, telemetry, or configuration writes. The root form
lists canonical commands/aliases and brief descriptions; scoped form includes:

- Argument/flag types, defaults, aliases, enums, required and conflicting flags.
- Auth requirements, default selection rule, and ID scope/provenance.
- Actual effects: portal read/write, local downloads/config, optional exports.
- Success data schema, common envelope/error schema, possible kinds/exit codes.
- Examples valid for this release and links to prerequisite discovery commands.

Read real flag metadata from the constructed Cobra tree. Add typed descriptors
only for semantics Cobra does not contain (auth/effects, relationships, result
model, constraints). The same descriptors validate inputs and document them.
Use one explicit JSON Schema definition/builder per DTO and validate fixture
results against it; avoid runtime schema inference from arbitrary `map[string]any`.
Register everything with its handler so missing schema/handler entries fail tests.

Selection errors return typed options (`id`, `label`, `scope`, optional legacy
`index`), the required flag, parent context, and a canonical retry argv. Unknown
IDs return `invalid_selection`; stale choices can include a list command.
Bound large choice sets; if truncated, say so and provide complete discovery.
Dynamic portal choices do not belong in the static schema command.

## 7. Migration coverage

| Wave | Commands/files | Extraction work |
| --- | --- | --- |
| A | `profile`, `semesters`; `features/profile.go`, `helpers/sem-details.go` | First full vertical slice; reuse `fetchStudentDetails`, separate fetching from printing/picking |
| B | `attendance`, `marks`, `grades`, `cgpa`; academic feature files | Numeric/raw fields, parse failures, semester scope/defaults; correct existing debug-dependent failure paths |
| C | `exams`, `timetable`, `today`, `tomorrow`, `dayafter`, `holiday`, `calendar`; schedule feature files | Error-returning calendar helpers, identity-preserving timetable parsing, typed advice, partial data, separate ICS effects |
| D | `hostel`, `library-dues`, `receipts`, `msg`, `events`; corresponding files | Replace string tables with domain records; return registration effects/results only when explicitly requested today |
| E | `course-allocation`, `course-page`, `course-page-archive`, `syllabus`, `da`; course/download files | Separate list/select/fetch; namespaced IDs; typed material/assignment/allocation results and operation failures |
| F | `facility`, `leave`, `nightslip`; service files | Reuse existing error-returning operations, eliminate duplicate printing; typed options/status/application outcomes; keep existing confirmation/apply behavior |
| G | `login`, `logout`, root flags, `proxy sync` | Local-operation results, prompt policy and errors; homogeneous per-operation aggregation |

All eligible public commands need truthful errors, prompt control, and a typed
schema before declaring 1–4 complete. Staged rollout may expose supported modes
per command, but must explicitly reject JSON on an unmigrated command instead
of presenting captured tables as the final typed contract. Completion/help are
documented format exceptions. Structured per-file filtering/rich download UX,
new write verbs, dry runs, and a distributable skill are separate follow-up work.

## 8. Proxy compatibility

Keep the existing invocation and response fields for current integrations:
`command`, `version`, timestamps/duration, `reg_no`, `flags`, `output`,
`structured_data`, and old error-kind spellings where required. Do not make the
legacy envelope the new public DTO.

- Parse flags using the same registered definitions as public commands, with a
  small translation layer for existing proxy-only aliases (including nightslip
  inputs and `sync`). Eliminate permissive map parsing and duplicate feature
  switches after migration; reject unknown flags before login.
- Both entry points invoke the same typed operation. Legacy tables/prose are
  rendered from returned data into a local buffer, not obtained by replacing
  process stdout. Keep table hooks only where compatibility tests need them.
- `sync` authenticates once, keeps per-command results, and reports a nonzero
  partial-failure exit while retaining useful data. Validate its complete
  command list/flags before authenticating or running any operation.
- Correct legacy exit statuses too. This is an intentional behavioral fix,
  documented in release notes; the current smoke test for empty credentials
  incorrectly assumes process success and must change.
- Keep legacy selection defaults at the proxy compatibility boundary where
  integrations rely on them, and test any difference explicitly. New JSON/table
  handlers share documented command rules rather than proxy-mode side effects.

## 9. Implementation sequence and review boundaries

1. **Contract and testable command construction:** introduce error/result/policy
   types, `NewRootCommand`, lazy config initialization, streams and the central
   exit path; metadata registry starts here. Preserve the visible CLI/aliases.
2. **Auth, transport, and policy failures:** typed login/home/session errors;
   preserve request status/cause in both transport paths; no-print validation;
   bounded refresh and cancellation; kill-switch/manual-input outcomes.
3. **First usable public slice:** implement `profile`, `semesters`, public
   flags, shared rendering, offline schema/catalog, and recovery errors. Prove
   parsing, auth, data, output and process status end to end before porting more.
4. **Academic reads:** wave B, stable semester selectors, typed assessment
   results, and parser/empty/fallback coverage. Extend schema with each handler.
5. **Schedule and planners:** wave C including calendar error propagation,
   identity preservation, deterministic output, and no incidental JSON exports.
6. **Remaining commands:** waves D–G, selector discovery, compatible service
   interactions, then remove legacy stdout capture/dispatch as coverage allows.
7. **Finish compatibility and documentation:** proxy wire fixtures, error-code
   release notes, complete command schemas/examples, and migration guidance.

Each slice includes its own tests and documentation. The foundation can merge
before feature migration, but no slice can claim JSON support until underlying
errors and data have been converted. Avoid a single PR rewriting all features.

## 10. Validation and completion criteria

Reuse `tests/feature_simple_e2e_test.go`, `feature_academic_e2e_test.go`,
`feature_planner_e2e_test.go`, `feature_download_e2e_test.go`, and
`feature_interactive_e2e_test.go`; retain terminal regression assertions while
adding typed operation assertions over their strict mocked transports.

Change `tests/cli_e2e_test.go` helpers to return stdout, stderr and exit code,
with a separate convenience assertion for successful runs. Add full-contract
cases for invalid flags, expired login, all-empty results, failed requests,
selection choices, malformed numeric data, and partial results. Use injectable
services for successful command-level tests; retain executable tests for actual
exit/stream behavior. Tests must not rely on a globally installed CLI.

Required checks:

- One JSON document on stdout, no ANSI or duplicated errors, nonzero failures;
  diagnostics never contain synthetic credential/cookie/CSRF sentinels.
- `--flag=value` and `--flag value`, aliases, flags before/after the command,
  `--`, bad output formats, and missing required values behave consistently.
- Help/schema/completion and invalid syntax cause zero auth/network/config
  writes; JSON errors also work before command dispatch.
- Closed stdin and every prompt-capable workflow terminate within a short test
  deadline; JSON/non-interactive flags never imply submission confirmation.
- Reorder fixture options and verify exact IDs target the same portal payload.
  Duplicate labels, wrong parent scope, conflicting flags, missing IDs, and
  consolidated/archive identity differences are covered.
- Typed data is independent of terminal width/debug/color, stable when source
  maps reorder, and validates against the advertised scoped result schema.
- Empty/absent/unparseable values are distinct; missing portal structure cannot
  silently become an empty success. Retain all existing attendance/planner rules.
- JSON schedule reads cause no ICS upload, unwanted file generation, browser,
  or folder open; explicit download requests still produce their intended files.
- Repeated fresh root construction does not retain flags or handlers; avoid
  parallel tests around remaining process globals until those paths are removed.
- Legacy proxy field/alias/selection behavior is fixture-tested, including
  `sync` partial results and corrected exit status.

Run targeted package/workflow tests for each slice, then `go test ./...` and
`go test -race ./...` for the completed integration. Keep performance tests in
their default mocked mode. No live portal mutation belongs in automated tests.

## 11. Live read-only validation

Static inspection and existing fixtures establish the implementation structure.
With the user's authorization, narrowly scoped live reads can verify current
semester option values, attendance/marks cell shapes, timetable identity, and
consolidated-course ID relationships. Record structural findings only; do not
commit account records, credentials, session tokens, or raw authenticated HTML.
Use an isolated temporary runner/config and prohibit submissions, downloads,
calendar uploads, OS open actions, and telemetry. Live results inform fixtures
and identity mapping; they do not replace mocked error-path tests.

Live check outcome: the official client policy endpoint returned normal mode
(`0`). A single login through the checkout's existing `login.Login` returned
invalid/empty cookies, so the runner stopped before the authenticated home page
and all academic reads. The API did not expose a typed cause; this does not
establish whether credentials, CAPTCHA, transport, or parsing caused the
failure. No repeat login, submission, download, calendar upload, or account-data
capture was performed. The temporary credential file was consumed and deleted.

Consequently, ID mappings and live portal field shapes remain unverified. The
plan's code findings are based on static inspection and existing mock fixtures;
they are not claims of a successful live portal test. Improving login error
classification is part of implementation step 2, before relying on live checks.
