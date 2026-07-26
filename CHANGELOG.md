# Changelog

All notable changes to the `improve` skill are documented here.

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/); versioning
is [semantic](https://semver.org/spec/v2.0.0.html). The version lives in two
manifests — `.claude-plugin/plugin.json` and the `metadata.version` field in
`skills/improve/SKILL.md` — and both are bumped together on release.

## Unreleased

### Added

- README: a mermaid diagram of the three main flows — `/improve` (full audit), `plan`, and `execute` — at the top of "How it works".

## 2.2.0 — 2026-07-26

Everything below came out of one end-to-end run of the skill against a ~15k-line
Go auth library — six plans audited, planned, executed, reviewed and merged. A
review of the *merged* result then found 22 defects that the six individual plan
reviews had each been structurally unable to see. These changes close the gaps
that let them through.

### Added

- **`review-merged [<base>]` — a new invocation variant, and the largest gap this release closes.** `execute` reviews one plan against a clean base; nothing reviewed the *combination* before integration, and that omission accounted for every one of the 22 findings. The pass reviews the accumulated branches as a single change — on an integration branch the user creates, since the advisor never merges — and leads with the defects only visible there: a guard one plan adds running *ahead* of what an earlier plan's test asserts (leaving the test green while it no longer exercises the control it is named for), error paths two plans jointly made distinguishable, conflict resolutions (unreviewed code by definition), and a linter suppression from a tooling plan that permanently buries a defect another plan recorded. Documented in `closing-the-loop.md`.
- **A required falsifiability bar in every plan.** A table of *delete this → that test must fail*, one row per behaviour the plan claims to establish, which the executor fills in with the **observed failure output**. The template previously asked "would one of these tests fail if the change were reverted?" as prose inside the coverage bar; nothing made it a criterion, so nothing enforced it. In the source run three tests shipped that could not fail — two shadowed by a guard a later plan added ahead of them, one whose test double never validated the thing it existed to validate. The new section spells out how a row goes hollow: deletions that break the build test the compiler, incidental panics substitute for assertions, and a test can fail because an *earlier* guard rejected the request rather than the one under test. A row nobody can make fail is the finding, not a formality. (`plan-template.md`, with matching slots in the executor report format and the reviewer's checklist in `closing-the-loop.md`.)
- **Premise verification in the refutation pass.** `adversarial-verification.md` already named the failure mode — "verifying a finding is not the same as verifying the fix" — and stopped at the diagnosis. It now carries a remedy: for plans introducing an artifact the rest of the system must accommodate (a cookie, header, config field, schema change, required call order), dispatch a verifier that attacks the **fix's premise** rather than the finding, with a prompt that forces enumeration of every configuration and deployment shape the repo supports. It runs in Phase 4 as part of `review-plan`, once a plan exists to attack — the Phase 3 refutation pass cannot do this, because at Phase 3 there is no fix yet. Three HIGH findings in the source run were "correct in the default configuration, broken in every other one" — a class the finding-level pass cannot reach by construction.

### Changed

- **Phase 4 now prompts `review-plan`.** It existed only as an invocation variant, so nothing in the workflow ever suggested running it; across six plans it was never used once. It is now called for on any plan that adds a security control, changes a public contract, or introduces a new artifact — the plans whose premise is most likely to be wrong.
- **Stacked plans get an explicit re-examination rule.** When plan N depends on plan M, the index records that M's tests must be re-*examined* after N lands, not merely re-run. "Still passing" and "still testing the same thing" are different questions, and the gap between them is where two of the source run's broken tests lived.
- **`execute` preconditions now include verifying the worktree's base commit.** A drift check compares `Planned at` against `HEAD` and is meaningless if the worktree was branched from somewhere else entirely — a failure that presents *exactly* like genuine drift, so the executor correctly stops and a full dispatch is wasted. Host worktree tooling may branch from the repo's default branch, and `origin/HEAD` is often stale; in the source run it pointed 63 commits back. Check and remedy are one command each. Stacked plans get the related note that their base is the dependency's commit, not their own `Planned at`.
- **Plan delivery to executors offers a second option.** Inlining the full plan is still always correct, but passing the absolute path in the main checkout is now documented for same-filesystem dispatches — read verified beforehand, path marked read-only. At ~30 KB per plan across a queue the difference is material.
- **The "deliberately not covered" section carries a warning.** It exists to separate an oversight from a decision, which quietly implies that any decision written into it is sound. In the source run the single riskiest interaction in a security change was recorded there and shipped untested. A decision still has to be a good one.
- **Every dispatched subagent prompt now carries Hard Rules 4 and 6 explicitly** — the executor preamble, the `review-merged` reviewers (plus a read-only constraint: they read the user's integration branch, not a disposable worktree), and the premise verifier, which instantiates inside the Phase 3 refuter template rather than shipping a bare prompt with no constraints block and no repository root. Subagents don't inherit the rules; the README's "copied verbatim into every subagent" claim is now true rather than aspirational.

### Fixed

- **README drift against released content**: the example findings table had lost the `Evidence` column and showed bare grades where the skill mandates grade-plus-sentence; the plan-properties list had lost the maintenance note and still said "three properties" above six bullets. Nothing at runtime reads the README, which is exactly how it drifts.

## 2.1.0 — 2026-07-25

### Changed

- **Plugin renamed to `blackshield`.** Claude Code namespaces plugin skills as `plugin:skill`, so the command was `/improve:improve`; it is now `/blackshield:improve`. Install becomes `/plugin install blackshield@improve`; existing plugin installs need `/plugin uninstall improve` first, since the old name no longer resolves. The skill itself is unchanged, and `npx skills add` installs are unaffected — still `/improve`.

## 2.0.0 — 2026-07-25

First release of the [Blackshield](https://blackshield.pt) fork. Major rather than
minor because the audit playbook was reworked section by section and several
outputs changed shape — findings now carry a graded Impact, plans require a test
plan and a code review, and dependency choices are surfaced as maintainer
decisions instead of recommendations. Anything built on 1.0.0's output format
should expect to adjust.

### Added

- **Adversarial finding verification** (`references/adversarial-verification.md`): fresh-context subagents prompted to *refute* each finding, returning REFUTED / OVERSTATED / CONFIRMED, clustered by subsystem. Re-reading your own finding re-confirms the mechanism, not the impact you inferred from it. Verdicts are evidence rather than rulings — a verifier told to refute over-refutes — so downgrades get re-checked against the code.
- **Intent & design doc ingestion in recon**: ADRs, PRDs, `CONTEXT.md`, `DESIGN.md`, `PRODUCT.md`. Documented tradeoffs stop being re-flagged as findings, direction suggestions stay grounded in stated intent, and plans use the repo's own vocabulary. No-op when absent.
- **Stack-matching skill discovery in recon**: libraries are read out of the dependency manifests and any skills for them loaded *before* auditing, so findings are judged against a framework's real idioms rather than model priors — often the difference between a finding and a false positive.
- **Prior-backlog ingestion in recon**: the existing `plans/README.md` is read up front and its "considered and rejected" list passed into every audit subagent, so dismissed findings aren't re-discovered and re-argued each run.
- **Canonical finding-ID prefixes** (`BUG`, `SEC`, `PERF`, `TEST`, `DEBT`, `DEP`, `DX`, `DOC`, `DIR`), mapped to the plan template's `Category` values. Parallel subagents were coining their own, which made findings impossible to dedup or to match against prior rejections.
- **Audit-coverage record in the plans index**: per-run effort level, commit, what was and wasn't examined, and which findings went through the refutation pass — so a later session can tell "examined and clean" from "never looked at".
- **Claude Code marketplace installation** via `.claude-plugin/marketplace.json`, now documented in the README.
- **`AGENTS.md`** for contributors: file layout and load order, the invariants that must not be weakened, the cross-file couplings and per-section ownership boundaries that make contradiction this repo's characteristic bug, and consistency checks to run in place of a test suite.

### Changed

- **Repository and attribution updated for the Blackshield fork.** Install instructions pointed at the upstream repo and would have installed the unmodified skill; they now resolve to `blackshieldpt/improve`. Plugin and marketplace manifests, and the skill's `metadata.author`, name Blackshield with `upstream: shadcn/improve` recorded. `LICENSE.md` carries both copyright notices — the original is retained because MIT requires it, with modifications attributed separately. README gains a provenance note and a "What's different here" section.

- **Every audit category gained the discipline that keeps it honest**, because they fail in different ways. Security names the repo's trust boundaries first and rates impact by reachability and precondition. Performance requires scale evidence before a cost claim. Tech debt requires evidence the code is actually changing, and blesses "ugly but stable, not worth doing". Test coverage asks whether a test would *fail* if the code broke, not whether lines are covered. Dependencies treat staying put as the default. Direction states what a feature costs to own forever.
- **Correctness, security, and performance are no longer JS/TS- and web-shaped.** Examples came almost entirely from one ecosystem, which read as a grep list for that stack and as silence for every other. Each defect class is now stated language-agnostically and instantiated across Go, Python, Rust, Ruby, Java, and SQL, with the repo's own stack and tooling taking precedence over the examples.
- **Four security classes that were entirely absent**: cryptography (broken algorithms, encryption without integrity, key management, TLS and certificate validation, inbound signature verification), authentication as distinct from authorization, unsafe deserialization, and availability under resource exhaustion. Plus SSRF — which previously appeared only as an example of a *false* finding — an agent/LLM surface bullet that Hard Rule 6 required without the playbook offering anywhere to put it, supply-chain posture, in-repo infrastructure config, secrets in git history, and audit-trail absence.
- **Defensive posture** added to both correctness and security rather than as a tenth category: unstated preconditions and unasserted invariants on one side, controls that fail *open* and single controls between untrusted input and a sink on the other. Judged against the repo's own convention, not against a maximum.
- **Adversarial verification now runs on every HIGH and MEDIUM impact finding in any category**, not only security — security was just where the failure mode was first noticed, and an invented impact is likelier where impact is inferred rather than read. Impact accordingly gained a defined HIGH/MED/LOW scale, since both the trigger and the table ordering key on it.
- **Every plan now requires a test plan with a coverage bar, and a code review.** The test plan names the cases, the layer each belongs at, and — for a bug fix — a test seen failing at the plan's commit; the bar is behavioural, not a percentage. Code review has two halves: the executor's self-review before reporting, and what *this* change's reviewer should distrust. If the advisor can't name that, the plan isn't specified yet.
- **The plan template no longer presents one ecosystem's strings as the form to fill in.** Its worked example stays TypeScript, because placeholders teach nothing, but is labelled as an instance and carries a substitution table across five ecosystems. A plan telling a Go executor to run `pnpm test` fails on its first gate, and a weak executor will try to satisfy it rather than stop.
- **`branch`** resolves the default branch properly (`git symbolic-ref`, falling back to a local `main`/`master`) and scopes to uncommitted and untracked work as well as commits — which is the state a pre-PR audit actually runs against.
- **`execute`'s model choice is a rule, not a fixed name**: one tier below the advisor, the user's choice when they name one, otherwise the host's mid-tier model — and if that name stops resolving, the cheapest model that can still follow a multi-step plan rather than the cheapest available.
- **Options for the maintainer are presented separately from defects.** Direction suggestions and dependency choices are unranked and carry no default, and the advisor never proposes removing or replacing a dependency on its own initiative, because what decides that isn't visible in the repo.
- Security guidance reframed toward defensive maintenance — identify the pattern, explain production impact, describe remediation — while keeping the canonical vocabulary. A stale ADR is reported as decision drift rather than used to suppress a finding.
- Hard Rule 2's worktree exception now covers the setup a fresh worktree needs (dependency install, one build), not only verification commands.
- `plans/` vs `advisor-plans/` substitution is mandated in the reference files, not just in Hard Rule 1.
- `deep` audit-subagent cap raised 8 → 9 so "one per category" covers all nine; verifier subagents get a separate budget so the two aren't confused.
- Every playbook bullet leads with a bold label, and for each defect two categories could claim, the ownership boundary is now stated on both sides.

### Removed

- `examples/001-extract-shadow-config-resolution.md`, the sample plan. Pinned to an upstream commit from June, with excerpts that had drifted from the code they quoted — a stale example of a format whose whole premise is that excerpts match live code. The README section it illustrated keeps its prose description.

### Fixed

- **`execute` no longer fails every plan it dispatches.** The template listed the index update as an unconditional done criterion, the executor was told to skip it because the reviewer owns the index, and the reviewer was told to re-run every criterion — one guaranteed failure per dispatch, costing a revision round or a spurious BLOCK.
- **Recorded rejections were never read back.** Phase 3 wrote them to the index "so they aren't re-audited next run", but recon never read the index and the subagent prompts never carried them.
- **Audit coverage died with the session**, so a later run couldn't distinguish unexamined areas from clean ones.
- **README had drifted from the skill** across "How it works", "What makes the plans executable", "Hard rules", and its example table — describing Vet as only the re-read the skill now calls its weakest check, omitting two of six hard rules including the prompt-injection rule, and showing findings-table columns that no longer exist.
- `plans/` vs `advisor-plans/` contradiction in Hard Rule 1.
- The `Planned at` date comes from `date +%F` rather than being reconstructed from memory, and the fails-first test criterion is pinned to the plan's commit rather than to `HEAD`, which has moved by the time an executor reads it.

### Security

- Hardened against secret leakage: findings and plans carry `file:line` and credential type only — never the value, not even truncated — and always recommend rotation rather than deletion alone. Hard Rules 4 and 6 are copied verbatim into every subagent prompt, since subagents don't inherit them.
- `--issues` checks repository visibility and requires explicit confirmation before publishing any plan describing a vulnerability, credential location, or other sensitive finding to a public repo.

## 1.0.0 — 2026-06-10

Initial release.

- Advisor workflow: recon → parallel audit → vet → prioritize → plan, with the advisor never modifying source code.
- Nine audit categories: correctness, security, performance, test coverage, tech debt & architecture, dependencies & migrations, DX & tooling, docs, and direction.
- Reference files: `audit-playbook.md` (per-category checklist, finding format, prioritization rubric), `plan-template.md` (plan structure and index format), `closing-the-loop.md` (`execute`, `reconcile`, `--issues`).
- Effort levels `quick` / `standard` / `deep`, and the `<category>`, `branch`, `next`, `plan`, `review-plan`, `execute`, and `reconcile` invocation variants.
- `--issues` publishing of plans as GitHub issues, and a sample plan in `examples/`.
