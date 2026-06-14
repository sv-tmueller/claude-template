# Codebase review - 2026-06-14

**Verdict: changes-requested**

Full-repo review of the claude-template: the `.claude/` agent team and project
skills, the two bounded review workflows, the project configuration, and the
documentation, design specs, and plans. The machinery is solid and the
model-pinning discipline holds throughout. One must-fix blocks: in
`tm-review-codebase.js` the area ceiling is taken with `opts.areas || 24`, which
accepts any truthy value, so a non-numeric or negative `areas` argument silently
runs zero area workers while the run still looks normal and the critic can
return `approve`. The rest are wiring gaps in the kickoff and sync-template
prose, two path-traversal hardening items in the workflow scripts, stale skill
paths in the design specs and the prior review report carried over from the
`tm-` rename (commit 6940104), and documentation polish.

This review supersedes `docs/reviews/2026-06-13-codebase-review.md`. Several of
that report's should-fix items have since shipped (the scout-failure guard, the
`--force-with-lease` push, the tester check-suite stop, the explicit architect
`JOB:` contract, the `safeRef` character allow-list, the advisor resume
tiebreaker and stateless-resume rewrite, the `tester` frontmatter `tools:`).
What remains open is carried below, plus the new findings from this pass.

## Must-fix

### Bounded orchestration workflows

- **.claude/workflows/tm-review-codebase.js:49 (bugs)** - `const MAX_AREAS =
  opts.areas || 24` accepts any truthy value without checking it is a positive
  integer. A non-numeric string (`areas: 'ten'`) makes `MAX_AREAS` that string,
  so `allAreas.slice(0, 'ten')` coerces to `slice(0, NaN)` and returns `[]`:
  zero area workers run. `scoutFailed` stays `false` (the scout returned a valid
  map), every area overflows into `scoutDropped`, `ceilingReached` becomes
  `true`, and the critic consolidates only the architecture worker's findings
  while the run otherwise looks normal. A negative integer (`areas: -1`) is the
  same class: `slice(0, -1)` drops the last area into `scoutDropped` and the run
  proceeds with N-1 workers. Verified in node: `[1,2,3,4,5].slice(0,NaN)` is
  `[]`, `.slice(0,-1)` is `[1,2,3,4]`, `Number.isInteger('ten')` is `false`,
  `!!'ten'` is `true`. Fix: validate and clamp,
  `const MAX_AREAS = Number.isInteger(opts.areas) && opts.areas > 0 ?
  opts.areas : 24`, which rejects strings, floats, zero, and negatives and falls
  back to the default.

## Should-fix

### Project skills

- **.claude/skills/tm-kickoff/SKILL.md:24-26, 54, 84-86 (bugs)** - When kickoff
  dispatches the architect (SUB_PLAN in step 1 / line 54, SPLIT_PROPOSAL at
  lines 24-26, ARBITRATION at lines 84-86), it never tells the lead to prefix
  the message with the job type. architect.md:16 requires the caller to pass
  `JOB: SUB_PLAN`, `JOB: SPLIT_PROPOSAL`, or `JOB: ARBITRATION` as the first
  line; without it the architect has no job signal and its behavior is
  undefined. Fix: at each architect dispatch, instruct the lead to prefix the
  message with the matching `JOB: ...` line.
- **.claude/skills/tm-kickoff/SKILL.md:13-14 (bugs)** - The `label:<name>`
  argument form is accepted ("or `label:<name>` to select by label") but no step
  says how to turn the label into issue numbers before the Gate. Fix: add a line
  in the preamble - when the argument begins with `label:`, run
  `gh issue list --state open --label '<name>'` and treat the returned numbers
  as `$ARGUMENTS` before Gate.
- **.claude/skills/tm-kickoff/SKILL.md:69-71 (bugs)** - Step 5 forwards the
  tester's UNTESTED CLAIMS to the reviewer on the first review, but step 6's
  re-review after a fix round does not say to re-forward the claims from the new
  re-test. Each fix round produces a fresh tester report with possibly updated
  claims; the reviewer should see the current ones. Fix: in step 6 change
  "re-test, then re-review" to "re-test, then re-review (forwarding the tester's
  UNTESTED CLAIMS from the re-test)".
- **.claude/skills/tm-advisor/SKILL.md:113-116 (bugs)** - The next-batch backlog
  is "all open issues with no `Part of batch #` line in their body". A batch
  tracking issue (title `Batch: ...`) has no such line by design (it is the
  parent), so an open batch issue from another batch, or the current one if not
  yet closed, is selected as a candidate package. Fix: exclude issues whose
  title starts with `Batch:` from the backlog, for example add
  `--search 'NOT Batch: in:title'` alongside the existing filter.
- **.claude/skills/tm-advisor/SKILL.md:107-110 (bugs)** - The resume path
  confirms merges with `gh pr list --state merged` "against the batch's PRs",
  but `gh pr list --state merged` lists every merged PR in the repo, not just
  this batch's, and no `--search` scoping is given. On a busy repo the check is
  unreliable. Fix: check each batch PR individually, for example
  `gh pr view <n> --json state` and confirm `state == MERGED`.
- **.claude/skills/tm-sync-template/SKILL.md:21-22 (bugs)** - Step 1 reads
  `.claude/template-version` "here" (in the clone) to get the stamp, but the
  applied stamp is the SHA recorded in the local repo, not the clone's own HEAD.
  The three-way guards in section 2 already use `<stamp>` correctly against the
  clone, so the intent is the local file; the wording in step 1 is ambiguous.
  Fix: clarify that the stamp is read from the local repo's
  `.claude/template-version`, and the delta is computed in the clone as
  `git log --oneline <stamp>..HEAD` and `git diff <stamp> HEAD`.

### Role agents

- **.claude/agents/developer.md:18-21 (bugs)** - The fix-round / resume path runs
  `git fetch origin <branch>` then `git checkout --detach FETCH_HEAD` with no
  check that the fetch resolved a real ref. If the branch name is wrong or not
  yet pushed, `git fetch origin <branch>` errors, but FETCH_HEAD may still hold
  a value from an earlier fetch, so a retried or scripted sequence can detach to
  the wrong commit. Fix: verify the remote ref before detaching, for example
  `git ls-remote --exit-code origin <branch>` first, or fail fast with an
  explicit error if the fetch returns nothing.
- **.claude/agents/developer.md:27-28 (bugs)** - The fresh-package path runs
  `git remote set-head origin --auto`, which queries the remote and can fail on
  a network error, leaving `origin/HEAD` unset or stale so the following
  `git switch -c ... origin/HEAD` builds from the wrong base. The step gives no
  fallback. Fix: document the fallback - if `origin/HEAD` cannot be resolved,
  use the known default (`origin/main`) and note the deviation; or replace
  `set-head --auto` with `git fetch origin` plus an explicit `origin/main`.
- **.claude/agents/tester.md:28-31 (bugs)** - The unconfigured-suite guard fires
  only when every command line in "Useful commands" starts with `#`. If the
  section is absent entirely, the guard does not trigger and the tester falls
  through to step 3 (attacking the change) with no suite run and no clear
  failure. Fix: also emit `VERDICT: FAIL` with "check suite not configured" when
  the Useful commands section is missing, not only when it is all comments.
- **.claude/agents/tester.md:38-46 (tests)** - The report contract has no field
  pinning the verdict to a commit. If the branch is rebased or force-pushed
  between the tester run and when the verdict is read, the VERDICT may be applied
  to a different commit than the one verified. Fix: add a
  `COMMIT: <full SHA of FETCH_HEAD at checkout>` field so the verdict is pinned
  to an immutable ref.

### Bounded orchestration workflows

- **.claude/workflows/tm-review-codebase.js:45-48, 141 (security)** - The
  `safeRef` regex `/^[\w.~^\/\-]+$/` allows `..` sequences, so `path: '../../etc'`
  passes validation and is interpolated into the `scope` string (including the
  literal `git ls-files -- ${root}` the scout is told to run), letting a
  faithful agent traverse outside the repo. Real-world risk is low (the value is
  the operator's own CLI argument and the agent is sandboxed) but the guard
  should close it. Fix: reject `..` after the regex passes
  (`&& !value.includes('..')`), or normalize with `path.resolve` and assert the
  result is under the cwd.
- **.claude/workflows/tm-review-changes.js:23-24, 100 (security)** - Same vector:
  the `safeRef` for `base` allows `..` (`base: '../../etc/passwd'`), and the
  value is interpolated into the `diffHint` string with the literal
  `git diff ${base}...HEAD` the workers run. Fix: add `&& !value.includes('..')`
  to the guard, consistent with the `path` fix above.
- **.claude/workflows/tm-review-codebase.js:158 (bugs)** - `scoutFailed` is true
  only when the map is missing or `map.areas` is not an array. If the scout
  returns `{ areas: [], dropped: [] }` (a valid schema response), `scoutFailed`
  is `false`, zero area workers run, but `coverageNote` has no CRITICAL warning
  and `workersFailed` does not include the scout, so an empty-area run looks
  clean. Fix: when `areas.length === 0 && !scoutFailed`, append a note to
  `coverageNote` that the scout returned zero areas and the review is
  architecture-only.
- **.claude/workflows/tm-review-codebase.js:207-209 (architecture)** -
  `ceilingReached` is set whenever `scoutDropped` is non-empty, but
  `scoutDropped` is the union of the scout's own self-drop list and the script's
  `slice(MAX_AREAS)` overflow, and both produce the same "raise the cap"
  message. When only the scout self-dropped (stayed under `MAX_AREAS`), raising
  the cap may not be the right remedy. Fix: track script overflow
  (`allAreas.slice(MAX_AREAS)`) and scout self-drop (`map.dropped`) separately
  and word a distinct message for each cause.
- **.claude/workflows/tm-review-changes.js / tm-review-codebase.js:n/a (tests)** -
  Neither workflow has automated tests. `safeRef`, `parseArgs`, the
  `covered`/`dropped` partition, and the `scoutDropped` union all have right
  answers for fixed inputs, and the `MAX_AREAS` coercion and `safeRef`
  path-traversal bugs above were found only by manual reasoning. The runtime
  globals (`agent`, `parallel`, `phase`, top-level `return`) make the file
  unparseable by `node --check`, but the pure helpers can be extracted and unit
  tested. Fix: add a test file exercising `safeRef` and `parseArgs` with boundary
  inputs (empty, null, `..` string, negative, zero, non-numeric, valid) and the
  partition/union logic with mocked `reviews` arrays.

### Project configuration

- **.gitignore:3-5 (security)** - The ignore list covers `.env`, `.env.local`,
  and `.env*.local` but not non-local environment files (`.env.production`,
  `.env.staging`, `.env.development`, `.env.test`). Verified:
  `git check-ignore .env.production .env.staging .env.development` exits 1 (not
  ignored), so these are one `git add .` away from being committed. Fix: add
  `.env.*` (which subsumes the `.local` patterns), or add explicit entries for
  the environment-specific files.

### Architecture designs and plans

- **docs/superpowers/specs/2026-06-12-advisor-operating-model-design.md:4-5, 12,
  23, 26, 49, 83, 97, 102 (bugs)** - The spec still names the deployed skills
  `/advisor`, `/kickoff`, and `/grill-me` (and the path
  `.claude/skills/kickoff/SKILL.md`), but commit 6940104 renamed all project
  skills to the `tm-` prefix. A reader using this spec to locate or sync the
  skills looks for the wrong paths. Fix: update to `/tm-advisor`, `/tm-kickoff`,
  `/tm-grill-me`, and `.claude/skills/tm-kickoff/SKILL.md`.
- **docs/superpowers/specs/2026-06-12-advisor-operating-model-design.md:85
  (bugs)** - Decision 6 says the advisor "proposes the next batch from the
  backlog" but "backlog" is never defined in the spec; this is the root of the
  same gap previously flagged in the skill. Fix: define the backlog in Decision
  6 - open GitHub issues with no `Part of batch #` line (and, per the skill
  finding above, excluding `Batch:` tracking issues), ordered by dependency then
  creation date.
- **docs/superpowers/specs/2026-06-13-review-codebase-design.md:15, 31, 33, 163,
  167-168, 174 (bugs)** - The spec names the workflow
  `.claude/workflows/review-codebase.js` and the sync command `/sync-template`,
  and references `/review-codebase` and `/review-changes`. All were renamed with
  the `tm-` prefix. Fix: update to `tm-review-codebase.js`, `/tm-sync-template`,
  `/tm-review-codebase`, and `/tm-review-changes`.
- **docs/superpowers/specs/2026-06-13-review-codebase-design.md:150-152
  (bugs)** - The Report contract describes the coverage object as "areas
  reviewed, areas dropped, failed workers", but the shipped REPORT_SCHEMA also
  requires `ceilingReached` (boolean) and includes `suggestedNextAction`
  (string), both driving the partial-coverage callout. Fix: add both to the
  coverage description so a consumer built from the spec produces a complete
  object.
- **docs/plans/37-review-codebase.md:23, 43-46, 57-254, 348-354 (scope)** - The
  four Task 1 code blocks carry the superseded design (default ceiling 8, no
  `slice(0, MAX_AREAS)` clamp, no `ceilingReached`/`suggestedNextAction`, no
  `parseArgs`, no scout-failure guard, no `safeRef`), and the File structure /
  Task 3 sections still use the pre-rename paths (`review-codebase.js`,
  `review-changes.js`, `sync-template`) and still tell the reader to add
  `/reviews/` to `.gitignore`, which the top-of-file Amendment reversed. The plan
  flags the divergence in "Design updates after planning" but leaves the stale
  blocks in place, so a session resuming from them would re-implement the old
  design. Fix: replace the four code blocks with a pointer
  ("the shipped `.claude/workflows/tm-review-codebase.js` is the source of truth;
  do not re-implement from these blocks") and either strike the Task 3 gitignore
  step or annotate that the Amendment reversed it.
- **docs/reviews/2026-06-13-codebase-review.md:23-36 and throughout (bugs)** -
  Every workflow and skill path in the prior review report uses the pre-rename
  names (`review-codebase.js`, `review-changes.js`, `kickoff`, `advisor`,
  `sync-template`, `grill-me`). The report predates commit 6940104, so a
  developer acting on one of its findings looks up the wrong file. Fix: a
  search-and-replace pass updating all paths to the `tm-` prefixed names. (Lower
  urgency than the live specs, since the present report supersedes it, but the
  file is part of the audit record.)

## Nits

### Project skills

- **.claude/skills/tm-grill-me/SKILL.md:15 and
  .claude/skills/tm-to-issues/SKILL.md:46 (style)** - The attribution reads
  "Copyright (c) 2026 Matt Pocock". An upstream library's copyright year is the
  year the work was created, not the current year; 2026 looks like the current
  date copied into a third-party notice. Fix: verify the year on the upstream
  `mattpocock/skills` repo and correct it, or drop the year if it cannot be
  confirmed.
- **.claude/skills/tm-advisor/SKILL.md:64-80 (scope)** - Section 4 (Run)
  re-describes much of the kickoff per-package pipeline (concurrent cap, parking,
  mirroring) with small differences, creating a second source of truth that can
  drift. Fix: reduce section 4 to "run packages through the kickoff per-package
  pipeline per `.claude/skills/tm-kickoff/SKILL.md`, with these differences:
  [the diff list]" and keep only the genuine deltas inline.
- **.claude/skills/tm-advisor/SKILL.md:87-88 (style)** - The closing chat line
  ("merge these PRs, then invoke /tm-advisor again ...") duplicates the wave-end
  sign-off phrase in tm-kickoff. Low severity in a skill doc, but if consistency
  matters the standard sign-off could live in one place.
- **.claude/skills/tm-kickoff/SKILL.md:70 (architecture)** - Step 6 says "the
  same fix loop with the must-fix findings" without naming the step it mirrors; a
  resuming reader must trace back to step 4. Fix: make the antecedent explicit
  ("send the must-fix findings to a fresh developer dispatch in the same format
  as step 4, then re-test, then re-review").
- **.claude/skills/tm-sync-template/SKILL.md:8-9, 47 (style)** - "judgment merge"
  is a non-standard compound used as a noun on line 9 and adverbially ("by
  judgment") on line 47. Fix: normalize to one plain phrase in both places, for
  example "a manual, file-by-file merge guided by the diff".

### Role agents

- **.claude/agents/architect.md:19-25 (scope)** - The SUB_PLAN job restates the
  checkpoint-bullet format already defined in CLAUDE.md. Fix: reference it
  ("produce checkpoint bullets per the sub-plan format in CLAUDE.md").
- **.claude/agents/architect.md:36 (style)** - "Anchor on the issue text and the
  four principles" uses the borderline-cliche "anchor on". Fix: "Decide using the
  issue text and the four principles in `~/.claude/CLAUDE.md`".
- **.claude/agents/developer.md:3 (style)** - The description mixes the agent's
  identity with dispatch conditions ("Use once per work package during
  /tm-kickoff fan-out ..."). Fix: keep the description to the identity
  ("Implements exactly one GitHub issue end to end: branch, TDD, conventional
  commits, draft PR.") and leave dispatch conditions to the orchestrator skill.
- **.claude/agents/developer.md:15-35 (scope)** - Steps 2 (fix round / resume)
  and 3 (fresh package) duplicate most of the fetch / detached-work / refspec
  logic, differing only in the branch condition, so the two paths can drift. Fix:
  collapse into one pattern - always `git fetch origin <branch or default>`; work
  detached from `FETCH_HEAD` if the branch exists, else
  `git switch -c feat/<n>-<slug> origin/HEAD`; the push refspec is identical.
- **.claude/agents/developer.md:45 (scope)** - "Touch only what the issue
  requires" restates the surgical-changes principle the developer already
  inherits from `~/.claude/CLAUDE.md`, which CLAUDE.md says agent files should not
  repeat. Fix: remove it or replace it with a project-specific nuance.
- **.claude/agents/tester.md:11 (style)** - "Throwaway scripts go in /tmp" is
  buried in the opening prose after the read-only statement. Fix: group it with
  the no-Edit/no-Write guardrails so the behavioral rule is visually distinct.
- **.claude/agents/reviewer.md:8, 13 (architecture)** - Two diff commands are
  offered (`gh pr diff <n>` and the `git diff` form) with no guidance on which to
  prefer; they can differ depending on GitHub's merge base versus local
  `origin/HEAD`. Fix: prescribe one per situation - prefer `gh pr diff <n>` when
  a PR number is given, fall back to the `git diff` form only when no PR exists.
- **.claude/agents/reviewer.md:25 (style)** - "verifiable behavior" sits in the
  quality-criterion list but is vague and overlaps the tester's role; the
  "weakened or deleted test is always a blocking finding" line already covers the
  actionable part. The other principles are also referenced by their canonical
  names ("simplicity first", "surgical changes") while this one deviates from
  "goal-driven execution". Fix: drop "verifiable behavior" (or rename it to the
  canonical principle) and rely on the specific blocking-finding sentence.

### Project configuration

- **.claude/template-version:1 (bugs)** - The file contains the literal
  `unknown`, so tm-sync-template runs in unknown-base mode on the first sync
  (every template file treated as a conflict, no three-way merge), exactly the
  degraded path the skill documents at SKILL.md:26-27. NEW-PROJECT-SETUP.md
  step 2 already directs the user to stamp the SHA before the first sync, so this
  is a usability rough edge, not a correctness bug; the first sync is noisier
  than necessary until the stamp is written. Fix: ship the template HEAD SHA at
  release, or note in the file that `unknown` means "stamp not yet written".
- **.claude/settings.json:1-5 (scope)** - The file holds only `enabledPlugins`,
  which is correct and minimal, but nothing marks which keys are template-owned
  versus project-owned even though tm-sync-template merges by key and CLAUDE.md
  references `defaultMode: acceptEdits`. JSON has no comments, so document in
  CLAUDE.md or NEW-PROJECT-SETUP.md that `enabledPlugins` is template-managed and
  `permissions`, `hooks`, `env`, `defaultMode` are project-owned.
- **.gitignore:n/a (architecture)** - `docs/reviews/` is tracked (per the
  2026-06-14 amendment that reversed the original `/reviews/` ignore) but there
  is no in-file comment recording the decision, so a future editor might re-add a
  `reviews/` entry. Fix: add a short comment noting `docs/reviews/` is tracked
  documentation and intentionally not ignored.

### Project and session documentation

- **CLAUDE.md:24-30 (architecture)** - "Where decisions live" references
  `docs/architecture/` and `docs/operations/`, which do not exist in the template
  (created per NEW-PROJECT-SETUP.md). The Repo layout section annotates them with
  "(see NEW-PROJECT-SETUP)" but this section does not. Fix: add the same
  parenthetical caveat to each non-existent entry here.
- **CLAUDE.md:144-165 (style)** - The Operating model section uses "CEO" once in
  its prose context without defining it in CLAUDE.md (the design doc defines it
  as the user). Fix: replace "CEO" in the body with "the user" or
  "the user (acting as decision-maker)".
- **CLAUDE.md:87-93 (scope)** - The Code style section keeps three placeholder
  examples flagged "to adapt or delete" (strict typing, money/units types, i18n).
  A session may read them as concrete policy. Fix: replace with a single note to
  add project-specific rules and remove the placeholder before the first
  implementation session.
- **CLAUDE.md:274 (scope)** - "(Add project-specific traps here ...)" is a filler
  placeholder inside the What-not-to-do list that a session reads as an
  instruction. Fix: remove it or wrap it in an HTML comment visible only during
  setup.
- **NEW-PROJECT-SETUP.md:9 (style)** - The `gh repo create` command omits
  `--clone`, which the README version (line ~54) includes; without it the user
  must clone manually before the "fill in CLAUDE.md placeholders" flow. Fix: add
  `--clone`.
- **NEW-PROJECT-SETUP.md:24 (architecture)** - Step 2 checks the agent team
  loaded (`/agents`) but adds no parallel check that the project skills
  registered, though they are equally prone to the silent install gap the step
  already references (claude-code#32606). Fix: add a check that tm-advisor,
  tm-kickoff, tm-grill-me, tm-to-issues, and tm-sync-template appear in the
  skills list.
- **NEW-PROJECT-SETUP.md:22 (style)** - `claude-code#32606` is an opaque
  shorthand with no URL, unresolvable by external users. Fix: use the full GitHub
  issue URL, or drop the number if the workaround command alone suffices.
- **NEW-PROJECT-SETUP.md:35-41 (style)** - Step 3 lists the docs tree to create
  (architecture, operations, plans, superpowers/specs) but omits `docs/reviews/`,
  which CLAUDE.md's repo layout (line 231) lists. The tm-review-codebase critic
  creates the directory itself (`make the docs/reviews/ directory if it does not
  exist`), so the first review does not fail; this is a consistency gap with the
  layout, not a runtime break. Fix: add `docs/reviews/` to the step 3 list.
- **docs/plans/37-review-codebase.md:7 (scope)** - The plan header still says
  "(default 8)" while the shipped default is 24. Fix: update to "(default 24,
  configurable via args.areas)".
- **docs/superpowers/specs/2026-06-13-review-codebase-design.md:143-144
  (style)** - The Report contract says "grouped by severity then area" while the
  shipped consolidate prompt and the report layout group by area first. Fix:
  align the spec and the prompt on one phrasing
  (severity sections, each organized by area).
- **docs/reviews/2026-06-13-codebase-review.md:10 (style)** - "robustness gaps"
  uses "robustness", on the project's banned AI-cliche list. (Carried as a nit;
  the present report supersedes that file.) Fix: replace with a concrete phrase
  such as "reliability gaps" or "error-handling gaps in agent and skill prose".

### Bounded orchestration workflows

- **.claude/workflows/tm-review-codebase.js:225 (architecture)** -
  `${JSON.stringify(suggestedNextAction)}` embeds the string with literal
  surrounding quotes, so the agent receives `"Coverage is partial: ..."`. Every
  other string in the prompt uses a bare template literal. Fix: use
  `${suggestedNextAction}` and keep `JSON.stringify` only for the arrays and the
  boolean.
- **.claude/workflows/tm-review-codebase.js:225 (architecture)** - The
  consolidate prompt says findings are grouped "by area" while the spec says
  "grouped by severity then area", which buries must-fix findings inside each
  area. Fix: align the prompt and spec (severity sections - must-fix, then
  should-fix, then nit - each organized by area).
- **.claude/workflows/tm-review-codebase.js:51 (architecture)** - `FINDING`,
  `FINDINGS_SCHEMA`, and the `dismissed` sub-schema are duplicated verbatim
  between the two workflows, diverging only in the `area`/`dimension` fields. The
  runtime has no shared imports, so the duplication cannot be removed, but there
  is no comment marking the shared base. Fix: add a comment in each file noting
  the shared base and the intentional divergence so they stay in sync.
- **.claude/workflows/tm-review-changes.js:14 and tm-review-codebase.js:17
  (style)** - tm-review-changes refers to "a Fable or Opus session" (bare
  "Fable") while CLAUDE.md uses "Fable 5". Minor naming drift. Fix: align on the
  canonical name when these files are next touched.

## Coverage

Full review, no truncation.

- **Areas reviewed:** Project skills (`tm-advisor`, `tm-kickoff`, `tm-to-issues`,
  `tm-grill-me`, `tm-sync-template`); Bounded orchestration workflows
  (`tm-review-changes`, `tm-review-codebase`); Role agents (`architect`,
  `developer`, `tester`, `reviewer`); Project configuration and settings
  (`.claude/settings.json`, `.gitignore`, `.claude/template-version`); Project
  and session documentation (`CLAUDE.md`, `README.md`, `NEW-PROJECT-SETUP.md`);
  Architecture designs and implementation plans (`docs/superpowers/specs/`,
  `docs/plans/`, `docs/reviews/`).
- **Paths not covered:** none. `docs/architecture/`, `docs/operations/`, and
  `e2e/` do not exist yet (created per NEW-PROJECT-SETUP.md) and are flagged in
  the findings above.
- **Workers that failed:** none.
