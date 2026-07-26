# Closing the Loop — execute, reconcile, issues

The advisor's job doesn't end at the plan. This file covers the three follow-through flows: dispatching an executor and reviewing its work (`execute`), keeping the plan backlog alive (`reconcile`), and publishing plans where work gets picked up (`--issues`).

The founding rule survives unchanged: **the advisor never edits source code.** In `execute`, a *separate executor subagent* edits code in an isolated git worktree; the advisor dispatches, reviews, and renders a verdict — like a tech lead who doesn't push commits to your branch.

**Plans directory:** this file writes `plans/` throughout. If the advisor chose `advisor-plans/` instead, substitute that path everywhere below — index reads, executor instructions, and status writes.

---

## `execute <plan>` — dispatch and review

### Preconditions (check all before dispatching)

- The repo is a git repository (worktree isolation requires it). If not: stop and say so.
- The plan file exists and its dependencies show DONE in `plans/README.md`. If not: stop, name the missing dependency.
- Run the plan's drift check yourself. If in-scope files changed since `Planned at`, reconcile the plan first (see below) — don't hand a stale plan to an executor.
- **Instruct the executor to verify its worktree's base commit before anything else**, and give it the remedy. A drift check compares `Planned at` against `HEAD`; it is meaningless if the worktree was branched from somewhere else entirely, and that failure presents *exactly* like genuine drift — the executor dutifully reports "the code doesn't match the excerpts" and stops, having burned a full dispatch. Host worktree tooling may branch from the repo's default branch rather than the branch you are working on, and `origin/HEAD` is often stale. The check is one command (`git rev-parse --short HEAD` equals the plan's `Planned at`); the remedy is `git checkout -B <plan branch> <planned-at SHA>` inside the worktree, which works because worktrees share the object database. Checking out the *branch* you are on will be refused — create a new one at the commit.
- **When plans stack, the base is the dependency's commit, not the plan's `Planned at`.** If plan N depends on plan M and M is not yet integrated, N's executor must branch from M's commit, or the files M created will not exist. Say so explicitly, and tell it that the plan's own drift-check command will therefore show M's changes — expected, not drift. Give it a narrower command that proves no *unrelated* file moved.

### Dispatch

Spawn **one** `general-purpose` subagent with `isolation: "worktree"`.

**Executor model**: one tier below the advisor — the skill's economics depend on the expensive model planning and a cheaper one executing. Use what the user named if they named one (`execute 003 haiku`). Otherwise default to the host's mid-tier model (in Claude Code as of this writing, `sonnet`); if that name doesn't resolve, pick the cheapest model that can follow a multi-step plan, not the cheapest available.

The subagent prompt must contain:

1. **The plan text.** The worktree contains only committed files, so if `plans/` is uncommitted the executor cannot read it from *its own* checkout. Two ways to deliver it, and the choice matters once you are dispatching more than one or two:
   - **Inline the full text.** Always correct, works everywhere. Costs the whole plan in every prompt — a real consideration at ~30 KB per plan across a queue.
   - **Pass the absolute path in the main checkout** (`/abs/path/repo/plans/003-x.md`) when the executor runs on the same filesystem. Confirm the file is readable before dispatching, tell the executor the path is in the *main* checkout rather than its worktree, and mark it read-only — it must never write there. If the read fails, fall back to inlining.

   Never assume the executor can find the plan on its own.
2. The executor preamble:

> You are the executor for the implementation plan below. Follow it step by
> step. Run every verification command and confirm the expected result before
> moving on. Touch only the files listed as in scope. If any STOP condition
> occurs, stop immediately and report. Do not improvise around obstacles.
> Commit your work in the worktree following the plan's git workflow section.
> One override: SKIP the plan's instruction to update `plans/README.md` —
> your reviewer maintains the index. Before reporting, audit every claim in
> your report against an actual tool result from this session — only report
> what you can point to evidence for; if a verification failed or was
> skipped, say so plainly. When finished, reply with exactly the report
> format below.

3. The report format:

```
STATUS: COMPLETE | STOPPED
STEPS: per step — done/skipped + verification command result
STOPPED BECAUSE: (only if STOPPED) which STOP condition, what was observed
FILES CHANGED: list
SELF-REVIEW: the outcome of each item in the plan's "Code review" self-review
             list — not "done", but what you found and what you did about it
TESTS: which cases from the plan's test plan you covered, at which layer, and
       (for a bug fix) confirmation the test failed before the change
FALSIFIABILITY BAR: every row of the plan's table — the deletion you made, the
       test you ran, and the ACTUAL failure output. Mark any row you could not
       make fail; that is a finding, not a formality. (Omit this section only if
       the plan has no falsifiability bar.)
NOTES: anything the reviewer should know (deviations, surprises, judgment calls)
```

### Review (the advisor's real job here)

Note on fresh worktrees: they share git history but **not installed dependencies or build outputs** — whatever the ecosystem calls them (`node_modules`, `.venv`, `target/`, `vendor/`, a module cache). The executor has to install first, and any check that resolves from build output may need one build even though the plan's command table, recon'd in the main tree, didn't mention it. Expect this; it isn't a deviation.

Review like a tech lead reviewing a PR against the spec — never fix anything yourself:

1. **Re-run every done criterion** in the worktree. Don't trust the executor's report — verify. One exemption: the plan's "`plans/README.md` status row updated" criterion does not apply — you told the executor to skip it and you own the index. Never fail or revise an executor over it.
2. **Scope compliance**: `git -C <worktree> diff --stat` against the plan's in-scope list. Any file outside scope fails review, full stop.
3. **Read the full diff.** Judge it against "Why this matters" (does it solve the actual problem?) and the repo conventions named in the plan (does it look like the rest of the codebase?).
4. **Audit the new tests** against the plan's test plan, not just its done criteria. Executors game criteria: a test that asserts nothing meaningful still turns the suite green and proves nothing. Read what each test actually asserts, check the cases named in the plan are the cases covered and at the layer specified, and for a bug fix confirm the claim that the test failed before the change — re-run it at the plan's `Planned at` commit if the report doesn't evidence it.
5. **Audit the falsifiability bar the executor reported**, starting with the rows carrying the plan's security or correctness claim. This is the highest-value thing in the review and the cheapest to skip. You are judging the reported output, not redoing the deletions — never edit source, in the worktree or anywhere else. For each row: is there actual output rather than a checkmark, does it name the test the row names, and does it fail *for the stated reason*? Two things to watch:
   - **An incidental failure is not a passing row.** A nil-dereference or a build error where the row's assertion should be means the test may still be asserting nothing — the crash is doing the work. The plan tells the executor to prefer reverting to the pre-fix behaviour, which produces a clean assertion failure; if the report shows a crash instead, send the row back rather than accepting it.
   - **Verify the row is testing what its name says.** A test can fail on deletion because an *earlier* guard rejects the request, not the one under test. If the plan added a guard ahead of an existing one on the same path, read the handler and confirm the later control is still reachable.

   Any row that is missing, unquoted, or fails for the wrong reason is REVISE feedback — name the row and what the output has to show. A row the executor reports it could not make fail is a finding about the plan, not an executor failure: the test asserts nothing, so refine the plan rather than sending it back. **If the plan predates the falsifiability bar and has no table**, skip this step, judge the tests on step 4 alone, and never fail an executor for a section its plan never contained; if the plan is security- or correctness-critical, reconcile it to add the table before the next dispatch.

### Verdict

**Documented deviations are judged on merit, not reflex-blocked.** "Do not improvise" exists to stop silent drift; an executor that hits a real obstacle (e.g. the plan's approach breaks existing test mocks), adapts minimally, and explains it in NOTES has done the right thing. Approve it if the adaptation serves the plan's intent and stays in scope; treat *undocumented* deviations as review failures.

| Verdict | When | Action |
|---|---|---|
| **APPROVE** | Criteria pass, scope clean, quality holds | Update index status to DONE. Present to the user: diff summary, worktree path and branch, anything from NOTES. **Merging is the user's decision — never merge, push, or commit to their branch.** |
| **REVISE** | Fixable gaps | SendMessage to the same executor with specific, actionable feedback ("criterion 3 fails: X; the error handling in `api.ts:90` swallows the error — use the Result pattern per the plan"). **Max 2 revision rounds**, then BLOCK. |
| **BLOCK** | STOP condition hit, scope violated unrecoverably, or revisions exhausted | Mark BLOCKED in the index with the reason. Refine or rewrite the plan with what was learned. Tell the user what happened and what changed in the plan. |

Running verification commands inside the executor's worktree is fine — it's isolated and disposable. The no-mutating-commands rule protects the user's working tree, not the worktree.

---

## `review-merged [<base>]` — review the combination before integrating

Every `execute` review sees one plan against a clean base. That is the right unit
for judging whether a plan was implemented, and the wrong unit for judging whether
the *result* is correct — because the defects that survive to this point are
precisely the ones no single-plan review can see.

Run it after the last `execute` and before the work reaches the user's branch.

### The commit to review

The pass needs one commit that contains every executed plan — call it `<tip>`.
Each `execute` leaves its own worktree branch, so no such commit exists yet, and
**you do not create it**: Hard Rule 1 forbids merging, and an integration branch
is the user's call. Ask for one and hand over the commands:

```
git checkout -b <integration branch> <base>
git merge <plan branch>          # one per executed plan, in the index's order
```

`<base>` is the commit the plans were planned at — the `Planned at` line, which is
normally identical across them. If they were planned at different commits, use the
earliest and say so in the report; the diff will then also contain unrelated work.

Then review `git diff <base>..<tip>` as **one change**, the way a maintainer would
review a release branch — not as N plans re-read in sequence.

Two variations: if the user already merged into a branch of their own, `<tip>` is
that branch and `<base>` the pre-merge commit — late is far better than never. If
they'd rather not build an integration branch at all, say the pass can't run
without one rather than falling back to reading the branches one at a time; that
is what `execute` already did, and it is the exact blindness this pass exists for.

**Ask which merges hit conflicts** while they were building `<tip>`, and record
the answer — the resolutions are the one part of the diff no review has ever seen.

### What only shows up here

Lead with these; they are the whole reason the pass exists.

- **Guard shadowing.** Plan N adds a check on a path where plan M already asserts
  something. If N's check runs first and rejects with the same response, M's test
  passes without ever reaching the control it is named for. **Take every test the
  earlier plans added on a path a later plan touched, and re-ask "would this still
  fail if its control were removed?"** — not "is it green?". This is the single
  most common finding of this pass.
- **A new artifact meeting the deployment surface.** A cookie, header, or required
  call order introduced by one plan now has to survive every configuration the
  project supports — not just the default the plan was written against. Enumerate
  the config fields and mounting patterns and walk each one.
- **Responses that were meant to be identical and no longer are.** Two plans
  touching the same error path can make it distinguishable — an extra header, a
  different status, a changed body — turning a deliberately opaque rejection into
  an oracle.
- **Conflict resolutions.** Every conflict resolved while building `<tip>` is
  unreviewed code — it belongs to no plan, so no `execute` review ever saw it.
  For each one the user named, diff the resolved region against both sides and
  confirm nothing was dropped.
- **A fix in one plan that silences a finding in another.** Most often a linter
  suppression added by a tooling plan that now permanently hides a defect a
  different plan (or the original audit) had recorded.
- **Cross-plan duplication.** Two plans independently adding the same section,
  entry, or helper — harmless individually, wrong together.

### How

Fan out with fresh-context reviewers over the combined diff, clustered by concern
rather than by plan — the plan boundaries are exactly the lines you are trying to
see across. One reviewer on the security-critical production paths, one on the
tests (with "list every test that would pass with its feature deleted" as an
explicit deliverable), one on anything else the diff touches. Give them the diff
range and the fact that the code came from independent branches merged together;
that framing is what makes them look for interactions.

Then verify their findings yourself, as in Phase 3 — a reviewer told to hunt for
interactions will over-report couplings that are not real.

### Output

The same findings table Phase 3 produces, plus one explicit statement per plan:
**does it still do what its review said it did?** Anything that regressed becomes
a new plan, numbered into the existing sequence. Record the pass in the index's
audit-coverage block so a later `reconcile` can tell "reviewed as a whole" from
"reviewed one plan at a time".

## `reconcile` — keep `plans/` alive

Process what happened since the last session. Read `plans/README.md` and every plan file, then per status:

- **DONE** — spot-check that the done criteria still hold on the current HEAD (cheap ones only). Mark verified in the index. Don't delete plan files — they're the record.
- **BLOCKED** — read the reason. Investigate the underlying obstacle in the codebase. Either rewrite the plan around it (new number if the approach changed fundamentally, in-place refresh otherwise) or mark REJECTED with one line of rationale.
- **IN PROGRESS** (stale) — flag it to the user; an executor probably died mid-run. Check the worktree if one exists.
- **TODO** — run the drift check. If drifted: re-verify the finding still exists (it may have been fixed in passing), then refresh the "Current state" excerpts and `Planned at` SHA. If the finding is gone, mark REJECTED ("fixed independently").

Finish with a short report: what's verified done, what was refreshed, what's rejected, and what's executable right now.

---

## `--issues` — publish plans as GitHub issues

Modifier on any planning invocation (`/improve --issues`, `/improve security --issues`). The flag is the user's authorization to create issues — never create them without it.

1. Preflight: `gh auth status` succeeds and the repo has a GitHub remote. If either fails, write the plan files as normal and say why issues were skipped.
2. Visibility check: `gh repo view --json visibility`. If the repo is **public**, warn the user that issues are publicly visible and get explicit confirmation before publishing any plan that describes a security vulnerability, credential location, or other sensitive finding.
3. Show the list of titles about to become issues; confirm once if interactive.
4. Per plan: `gh issue create --title "<plan title>" --body-file <plan file>`. Labels: `improve` plus the category — apply only if the labels exist or can be created without erroring; skip labels rather than fail.
5. Record each issue URL in the plan's Status block (`- **Issue**: <url>`) and the index.

The plan file remains the source of truth; the issue is distribution. The self-containment rule pays off here — the issue body needs no edits to make sense to whoever (or whatever) picks it up.
