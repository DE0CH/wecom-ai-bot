---
name: owner-dev-rules
description: Owner's general dev mandates and preferences live in this repo's CLAUDE.md, ported from paper-trail-main 2026-07-15
metadata:
  type: feedback
---

The owner's standing dev rules — force-push ban, test immutability
("Test Deletion" protocol), strict TDD for bug fixes, flakes-are-bug-
reports, orchestrator-only 4GB machine (no local builds/tests; defer to
GitHub Actions), --no-ff merges, never reuse version numbers, CI-only
deployment, rbw-signed commits, worktree-per-agent, Edit/Write tools
only, plain-text formats, TypeScript/Python-over-shell taste, docs
style — are recorded in this repo's CLAUDE.md.

**Why:** These were distilled from ~/paper-trail-main (its CLAUDE.md and
memory/) at the owner's request on 2026-07-15; they are general owner
mandates, not Paper Trail specifics.

**How to apply:** Follow CLAUDE.md here; don't re-litigate without
asking. For the incident history behind a rule (why scrollbars must be
native, the v0.5.12 startup_failure, the update-404 spaces bug), consult
~/paper-trail-main/memory/. Related: [[wecom-bot-project]].
