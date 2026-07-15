# WeCom AI Bot — project instructions

Standing rules and preferences the owner established during Paper Trail
development (ported 2026-07-15 from ~/paper-trail-main). Follow them;
don't re-litigate without asking.

## Project context

- Only TWO people work on this project: the owner (developer) and one
  non-technical partner. The 企业微信 enterprise / legal entity exists
  solely for the two of them — there are no other stakeholders, so the
  owner can and should get FULL admin access to the 管理后台 rather
  than working through minimal-trust handoffs.

## Hard rules

- **NEVER force push** — any ref, any reason. History is append-only;
  fix forward.
- **Tests are IMMUTABLE contracts.** Never edit or delete existing test
  code without the owner's explicit permission — only ADD tests. "DRY
  does not apply to tests": duplicate suites rather than
  share/parametrize. Any commit with red (-) lines in a test file must
  start its message with "Test Deletion" and contain ONLY test changes.
  If a change requires a test to change: finish all requested work with
  the test failing, stop, ask.
- **Bug fixes follow strict TDD**: (1) reproduce the bug; (2) find the
  root cause; (3) write a regression test and WATCH IT FAIL on the bug;
  (4) fix; (5) watch it pass. If the test itself needs fixing
  afterwards, unapply the fix to see the corrected test fail, then
  re-apply.
- **Flakes are bug reports**: a sometimes-failing test gets
  root-caused and pinned with a NEW deterministic test (force the
  condition — synthesize the event, control the ordering). A rerun only
  unblocks the pipeline; the investigation is mandatory.
- **This machine is an orchestrator only** (4GB RAM — local builds and
  media processing have OOM-crashed it). Local shell use is limited to
  git, gh, file edits, and quick metadata. Defer builds, tests, and any
  heavy computation to GitHub Actions: edit, commit, push, then trigger
  and watch workflows with gh.
- Merge with `--no-ff`, never cherry-pick.
- **Never reuse a version number.** A version that failed to build gets
  skipped — no tag moving — and a "(skipped)" changelog note records why.
- **Deployment and secrets live in CI only**, never on a dev machine.
  Dev machines push commits and tags; CI deploys.

## Taste & stack

- TypeScript (strict) for apps; Python for scripts — never shell
  scripts. Frameworks and existing tooling over hand-rolled code.
- Existing/stock/native components over custom ones — hand-rolled
  substitutes silently lose polish and affordances. Only build custom
  when the platform truly offers nothing.
- User-facing file formats are plain-text, line-oriented, and
  git-diff-friendly — never JSON.
- Measure before optimizing; document limits (hard vs soft) instead of
  enforcing caps.
- Docs: README is a user-facing quick-start only; developer material
  goes in CONTRIBUTING.md. Full sentences (no fragments outside section
  titles), no development war stories, don't over-emphasize obvious
  points. Instructions the owner gives Claude are NOT contributor/user
  documentation.

## Git & collaboration workflow

- One commit per feature/fix, after its tests pass. Commit a draft
  before starting to debug.
- Commits are SIGNED via the rbw (Bitwarden CLI) ssh-agent — use the
  agent, never extract the key. Socket:
  `/run/user/$(id -u)/rbw/ssh-agent-socket` (exported in ~/.bashrc).
  Unlock with `rbw unlock` (check `rbw unlocked`). Gotcha: a shell with
  a stale SSH_AUTH_SOCK pointing at the dead Goldwarden socket fails
  with "No private key found" — override per-command with
  `SSH_AUTH_SOCK=/run/user/$(id -u)/rbw/ssh-agent-socket git commit …`.
- The owner pushes concurrently while Claude works: when a push is
  rejected, `git pull` (rebases per .gitconfig) and push again. Never
  force push to resolve it.
- **Assume several agents may share this working tree.** Never
  `git checkout <branch>` in the shared checkout for isolated work —
  each agent works in its own `git worktree`, and the orchestrator
  merges the branches back. Leave the shared checkout on its baseline.
- Edit files ONLY with the Edit/Write tools — never via python or shell
  scripts. If an edit fails with "file has been modified", the owner is
  probably editing concurrently: re-read and retry.
- Installing dependencies is pre-approved. gh CLI is authed (DE0CH).

## Working style

- Don't block on questions mid-work: proceed as best you can, ask about
  the choices afterwards if needed.
- Parallelize with agents and background tasks. Whenever a background
  command is expected to wake Claude on completion, ALSO set a 5-minute
  ScheduleWakeup alarm that does nothing but wake; the woken agent
  decides what to do.
- Final answers go at the END of the reply; acknowledge every mid-work
  user message in the very next reply.
- Correcting a record (memory, docs, comments): DELETE the wrong
  statement and state only the current truth — never keep the error and
  annotate it. Records hold the current fact, not the history of
  mistakes.
- 3-letter file extensions; high-entropy temp file names.
- Before any context compaction, capture the full session transcript
  (~/.claude/projects/<project>/<session>.jsonl, gzipped) into
  docs/transcripts/ and commit it. Naming: session-N.
- When a feature/fix lands, notify the owner (PushNotification) with
  the deliverable — don't batch.
- CI workflows must report ALL failures in a run — never fail fast —
  and keep per-suite verdicts visible as separate steps.
