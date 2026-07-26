# Handoff Plan Template

Every plan is written for an executor model that has **zero context**: it has not seen the advisor session, the audit, the other plans, or any prior conversation. It may be a smaller/cheaper model. Assume it is competent at following explicit instructions and weak at filling gaps, recovering from ambiguity, or knowing when to stop.

Three properties make a plan executable by a weaker model:

1. **Self-contained context** — everything needed is in the file: paths, code excerpts, conventions, commands.
2. **Verification gates** — every step ends with a command and its expected result. The executor never has to *judge* whether it succeeded.
3. **Hard boundaries and escape hatches** — explicit out-of-scope list, and "STOP and report" conditions instead of letting the model improvise when reality doesn't match the plan.

File naming: `plans/NNN-short-slug.md`, numbered in recommended execution order.

**The worked example below is one repo's instance, not a form to copy.** It happens
to be a TypeScript service using pnpm, because a template with placeholder strings
teaches nothing. **The structure is fixed; every path, command, filename, and tool
name is replaced with the audited repo's own** — verified during recon, never
guessed and never carried over from this file. A plan whose command table says
`pnpm test` for a Go repo fails on its first gate, and a weak executor will try to
make it work rather than stop.

Rough shape of the substitutions, to make the point concrete:

| Template says | Go | Python | Rust | Java | Ruby |
|---|---|---|---|---|---|
| `pnpm install` | `go mod download` | `uv sync` / `pip install -e .` | `cargo fetch` | `mvn -q dependency:go-offline` | `bundle install` |
| `pnpm typecheck` | `go vet ./...` / `go build ./...` | `mypy .` | `cargo clippy -- -D warnings` | `mvn -q compile` | `srb tc` (or none) |
| `pnpm test -- <filter>` | `go test ./pkg/... -run X` | `pytest -k X` | `cargo test X` | `mvn -q test -Dtest=X` | `bundle exec rspec path` |
| `pnpm lint` | `golangci-lint run` | `ruff check .` | `cargo fmt --check` | `mvn -q spotless:check` | `rubocop` |
| `src/orders/api.ts` | `internal/orders/api.go` | `orders/api.py` | `src/orders/api.rs` | `.../orders/Api.java` | `app/orders/api.rb` |

Not every repo has every row. Omit what doesn't exist rather than inventing it —
and if there's no static check or no test command at all, that's a finding for the
audit and a prerequisite plan, not a gap to paper over here.

**Plans directory.** This file writes `plans/` throughout. If the advisor chose
`advisor-plans/` instead (because `plans/` was already taken for an unrelated
purpose), substitute that path **everywhere** — the executor instructions, the
done criteria, the index heading, and every cross-reference in every plan you
write. A plan that points its executor at a directory that doesn't exist is
broken.

---

## Template

```markdown
# Plan NNN: <Imperative title — what will be true after this plan>

> **Executor instructions**: Follow this plan step by step. Run every
> verification command and confirm the expected result before moving to the
> next step. If anything in the "STOP conditions" section occurs, stop and
> report — do not improvise. When done, update the status row for this plan
> in `plans/README.md` — unless a reviewer dispatched you and told you they
> maintain the index.
>
> **Drift check (run first)**: `git diff --stat <planned-at SHA>..HEAD -- <in-scope paths>`
> If any in-scope file changed since this plan was written, compare the
> "Current state" excerpts against the live code before proceeding; on a
> mismatch, treat it as a STOP condition.

## Status

- **Priority**: P1 | P2 | P3
- **Effort**: S | M | L
- **Risk**: LOW | MED | HIGH
- **Depends on**: plans/NNN-*.md (or "none")
- **Category**: bug | security | perf | tests | tech-debt | migration | dx | docs | direction
- **Planned at**: commit `<short SHA>`, <YYYY-MM-DD>
- **Issue**: <GitHub issue URL — only when published via `--issues`; omit otherwise>

## Why this matters

2–5 sentences. The problem, its concrete cost, and what improves when this
lands. Written so the executor (and a human reviewer) understands the intent —
intent is what lets a correct judgment call happen when a detail is off.

## Current state

The facts the executor needs, inlined — never "as discussed" or "see audit":

- The relevant files, each with one line on its role:
  - `src/orders/api.ts` — order-list endpoint; contains the N+1 (lines 130–160)
- Excerpts of the code as it exists today (short, with `file:line` markers),
  enough that the executor can confirm it's looking at the right thing.
- The repo conventions that apply here, with a pointer to one exemplar file:
  "Error handling follows the Result pattern — see `src/lib/result.ts` and its
  use in `src/users/api.ts:40-60`. Match it."
- Any documented vocabulary or design constraints the plan must honor, inlined
  from the intent/design docs found in recon: the relevant `CONTEXT.md` terms
  the executor should use in names and comments, the `DESIGN.md` tokens/components
  to reuse, or the ADR whose decision this work must stay consistent with. Quote
  the specific lines — the executor has not read those docs.

## Commands you will need

| Purpose            | Command                  | Expected on success |
|--------------------|--------------------------|---------------------|
| Install / restore  | `pnpm install`           | exit 0              |
| Static check       | `pnpm typecheck`         | exit 0, no errors   |
| Tests (scoped)     | `pnpm test -- <filter>`  | all pass            |
| Tests (full suite) | `pnpm test`              | all pass            |
| Lint / format      | `pnpm lint`              | exit 0              |

(This repo's exact commands, verified during recon — not guessed, and not carried
over from the template. Drop rows the repo doesn't have. Note anything that needs
setup a fresh worktree lacks: a build before checks resolve, a container for
integration tests, a seeded database.)

## Suggested executor toolkit

(Optional. Fill this from the stack-matching skills recon actually found and
loaded — not from a guess about what might be installed. Skip the section when
recon found none.)

- Skills the executor should invoke, and for what — naming the actual skill recon
  loaded for this stack, e.g. "use the `<orm-name>` skill when rewriting the query
  in step 2" or "use the `<framework>` best-practices skill for the caching in
  step 3". If a skill informed a finding or a step here, name it: the executor is
  a weaker model and needs the same capability you had when you specified the work.
- Reference docs worth reading before starting, by path or URL.

## Scope

**In scope** (the only files you should modify):
- `src/orders/api.ts`
- `src/orders/api.test.ts` (create)

**Out of scope** (do NOT touch, even though they look related):
- `src/orders/legacy-api.ts` — deprecated path, scheduled for deletion;
  changing it wastes effort and risks the v1 clients still pinned to it.
- Any change to the public response shape — clients depend on it.

## Git workflow

(Filled from recon — match the repo's observed conventions.)

- Branch: `advisor/NNN-<slug>` (or the repo's branch-naming convention if one is evident)
- Commit per step or per logical unit; message style: <match repo, e.g. conventional commits — include an example from `git log`>
- Do NOT push or open a PR unless the operator instructed it.

## Steps

### Step 1: <imperative title>

What to do, precisely. Reference exact files/symbols. Include the target code
shape when it's load-bearing (the pattern to produce, not necessarily every
line).

**Verify**: `<command>` → <expected output>

### Step 2: ...

(Each step small enough to verify independently. Order steps so the codebase
is never broken between steps when possible — e.g. add new path, switch
callers, then remove old path.)

## Test plan & coverage

Required in every plan — including refactors, migrations, and docs changes, where
the answer may legitimately be "the existing suite covers this, and here is the
command that proves it".

- **Cases to cover, named individually**: the happy path, the specific
  bug/regression this plan addresses, and each edge case by name. "Add tests for
  the new function" is not a test plan.
- **The layer each case belongs at** — unit, integration, or e2e — choosing the
  cheapest layer that can actually catch the defect (see §4 of the audit
  playbook). Say which. Without a layer named, an executor writes whatever is
  easiest, which is usually a unit test mocking the thing that actually broke.
- **For a bug or regression fix, the test must be seen failing first**: state it as
  a criterion — `<test command>` fails at `<planned-at SHA>` with `<the specific
  failure>`, and passes after the change. A test written after the fix and never
  observed to fail proves nothing about the bug it claims to cover.
- **Structural exemplar**: "model after `src/users/api.test.ts`" — match the
  repo's existing test idiom (setup, fixtures, naming, assertion style) rather
  than importing one.
- **Coverage bar for the changed code**, stated as behaviour and not a
  percentage: which new or changed branches must be exercised, including the
  error paths.
- **What these tests deliberately do not cover**, and why — so the reviewer isn't
  left guessing whether a gap is an oversight or a decision. Be careful here: this
  section exists to separate an oversight from a decision, so a *decision* written
  into it still has to be a good one. An untested interaction that is the riskiest
  thing in the change does not become acceptable by being listed.
- **Verification**: `<test command>` → all pass, including the N new tests.

### Falsifiability bar

**Required.** A green suite proves nothing by itself — a test that cannot fail
passes for exactly the same reason a correct one does. Every plan states, as a
table, what must break when the work is removed:

| Delete / revert this | This test must fail | Observed failure (goes in the executor's report) |
|---|---|---|
| `the specific guard, line-referenced` | `TestName` | |
| `the specific call the fix adds` | `TestName` | |

Rules that make it real rather than ceremonial:

- **One row per behaviour the plan claims to establish.** If the plan adds a
  guard, a validation, and a call, that is three rows.
- **The executor must perform each deletion, run the test, capture the actual
  failure output, and restore.** Not "confirmed" — the output. This is a done
  criterion, and the output goes in the executor's **report**: the third column
  stays empty in the plan file, which the executor works from as text or as a
  read-only path and cannot write to.
- **The deletion must leave the code compiling.** A row that fails to build tests
  the compiler, not the test. Prefer neutering the behaviour (make the guard's
  branch unreachable, drop the option from the call) over deleting a symbol other
  code references.
- **Prefer reverting to the pre-fix behaviour over deleting outright** — that is
  what a regression actually looks like, and it produces a clean assertion failure
  instead of an incidental panic.
- **A row nobody can make fail is the finding**, not a formality to wave through:
  that test asserts nothing and the plan is not done. Say so plainly rather than
  quietly dropping the row.
- **Watch for guards that shadow each other.** If an earlier check in the same
  handler rejects the request with an identical response, a later test can pass
  without ever reaching the code it names. When a plan adds a guard to a path that
  already has one, add a row proving the *later* control is still reachable.

For a test double or fixture the plan introduces, the same bar applies to the
double itself: if deleting the thing the double is supposed to verify leaves every
test green, the double is inert and the coverage is imaginary.

## Code review

Two audiences. Both are part of the plan, not optional extras.

**Executor: self-review before you report.** Do these and state the outcome of
each in your report:

- Re-read the full diff against "Why this matters". Does it solve the stated
  problem, or does it only satisfy the done criteria? Those are different, and
  the second one passing is how a wrong change ships.
- Every hunk traces to a numbered step. Anything that doesn't is out of scope —
  revert it, even if it's an improvement.
- The new tests assert on observable behaviour, not on the implementation you
  just wrote. A test that mirrors the code will pass for a broken rewrite.
- Conventions match the exemplar file named in "Current state" — compare against
  it directly rather than from memory.
- Nothing left behind: debug output, commented-out code, stray TODOs, unrelated
  formatting churn, temporary files.
- Report honestly. Any verification you skipped or that failed, and any deviation
  from the plan with its reason. A documented deviation is judged on merit; an
  undocumented one is a review failure.

**Reviewer: what to scrutinize in this specific change.** Written by the advisor,
per plan — never boilerplate:

- `<the riskiest hunk and why — e.g. "the batching in step 2 changes failure
  semantics: one bad item used to fail one request, now it fails the batch">`
- `<the assumption most likely to be wrong, and how to check it cheaply>`
- `<what a fully green test suite would still not catch here>`

If you can't name what a reviewer should distrust, the plan is under-specified —
go back and find it before handing this over.

## Done criteria

Machine-checkable. ALL must hold:

- [ ] `pnpm typecheck` exits 0
- [ ] `pnpm test` exits 0; new tests for <X> exist and pass
- [ ] the regression test for <X> fails at `<planned-at SHA>` and passes now
- [ ] **every row of the falsifiability bar reported with its observed failure
      output**; any row that could not be made to fail is called out as such
- [ ] `grep -rn "<old pattern>" <the repo's source root>` returns no matches
- [ ] No files outside the in-scope list are modified (`git status`)
- [ ] Code-review self-check completed; its outcome is in the report
- [ ] `plans/README.md` status row updated — **not applicable if a reviewer
      dispatched you and told you they maintain the index**

## STOP conditions

Stop and report back (do not improvise) if:

- The code at the locations in "Current state" doesn't match the excerpts
  (the codebase has drifted since this plan was written).
- A step's verification fails twice after a reasonable fix attempt.
- The fix appears to require touching an out-of-scope file.
- You discover the assumption "<key assumption>" is false.

## Maintenance notes

For the human/agent who owns this code after the change lands:

- What future changes will interact with this (e.g. "if pagination is added
  to this endpoint, the batching in step 2 must be revisited").
- Any follow-up explicitly deferred out of this plan (and why).
- What the tests here will *not* catch if someone changes this later.

(What a reviewer should scrutinize now lives in "Code review" above; this section
is for whoever owns the code months from now.)
```

---

## Index file: `plans/README.md`

Written once by the advisor after all plans, updated by executors:

```markdown
# Implementation Plans

Generated by the improve skill on <date>. Execute in the order below unless
dependencies say otherwise. Each executor: read the plan fully before starting,
honor its STOP conditions, and update your row when done.

## Audit coverage

Recorded per run so a later session can tell "examined and clean" from "never
looked at". Append a new block per run rather than overwriting the last one.

- **Run**: <YYYY-MM-DD>, effort level `quick` | `standard` | `deep`, at commit `<short SHA>`
- **Audited**: <categories covered, and which packages/paths they were scoped to>
- **Not audited**: <categories skipped, packages excluded, and why — e.g. "docs,
  DX, direction skipped at `quick`; `packages/legacy/*` excluded as frozen">
- **Verification**: <which findings went through the adversarial refutation pass;
  say so if it was capped, and name the ones that are self-vetted only; which
  plans got a premise verifier>
- **Merged review**: <`review-merged` run over plans <NNN>–<NNN> at `<short SHA>`,
  or "not run — plans reviewed one at a time only">, so a later `reconcile` can
  tell the two apart

## Execution order & status

| Plan | Title | Priority | Effort | Depends on | Status |
|------|-------|----------|--------|------------|--------|
| 001  | ...   | P1       | S      | —          | TODO   |
| 002  | ...   | P1       | M      | 001        | TODO   |

Status values: TODO | IN PROGRESS | DONE | BLOCKED (with one-line reason) | REJECTED (with one-line rationale — finding fixed independently or approach abandoned)

## Dependency notes

- 002 requires 001 because <reason>.

## Findings considered and rejected

- <finding>: not worth doing because <one line>. (So nobody re-audits it.)
```

## Quality bar — check before finishing each plan

- Could a model that has never seen this repo execute this with only the plan file and the repo? If any step requires knowledge from the advisor session, inline that knowledge.
- Is every verification a command with an expected result, not a judgment ("make sure it works")?
- Does every step name exact files and symbols, not "the relevant module"?
- Are the STOP conditions specific to this plan's actual risks, not boilerplate?
- Does every path, command, and tool name come from *this* repo, with nothing carried over from this template's TypeScript example?
- Does the test plan name a layer per case, and — for a bug fix — a test that fails before the change?
- Does "Code review" name what a reviewer should distrust in this specific change, rather than generic advice?
- Would a reviewer reading only "Why this matters" + "Done criteria" understand what they're approving?
- No secret values anywhere in the file — locations and credential types only.
- "Planned at" SHA and date are filled in from actual `git rev-parse --short HEAD` and `date +%F` output, not guessed, and the in-scope paths in the drift check match the Scope section.
