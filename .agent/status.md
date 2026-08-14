# Build Session Status Log

## 2026-08-14 — Audit: GitHub credential isolation gap found and closed

**Trigger:** a routine submodule-pointer fix in this repo (bumping
`home-assistant/tariff-au` off an orphaned/rebased-away `ha-tariff-au` commit,
`95495fd` → `b9e0dc8`, since the orphaned SHA 404'd in GitHub's web UI while
still being fetchable via the Git Data API) landed as a commit authored
`axilotl <dev@openenergy.org.au>` — the founder's personal identity — instead
of `oec-agent`. Founder caught this immediately and asked for a full audit
of why, given this project's explicit goal of routing all GitHub-facing
actions through `oec-agent` (the founder's personal account carries an
active GitHub fraud/compliance flag; using a separate bot account to avoid
compounding that is intentional project posture, not incidental).

**Root cause — two independent, compounding bugs, not one:**

1. **SSH transport bypasses `gh`'s credential routing entirely.** This
   repo's `origin` (and `ha-tariff-au`'s) used `git@github.com:...` SSH
   remotes. SSH auth has nothing to do with which `gh` CLI account is
   active — it just offers local SSH keys against `github.com` in order.
   With no `~/.ssh/config` `Host` override, it silently fell back to the
   one general-purpose key (`~/.ssh/id_rsa`), which is registered to the
   founder's personal GitHub account. Confirmed directly: `ssh -T
   git@github.com` → "Hi axilotl!" — true even for pushes to `ha-tariff-au`,
   where commit *authorship* metadata correctly said `oec-agent` (and was
   even SSH-signed with a dedicated `oec-agent` signing key) but the actual
   push-transport authentication was the founder's personal account the
   whole time. Authorship/signing and transport-level push auth are
   separate mechanisms in git; only the second one determines which GitHub
   account performed the push, and it had been silently wrong all session.

2. **This repo had no local `git config user.*` override at all**, so
   commits made here fell through to the global config chain:
   `~/.gitconfig` (`axilotl <dev@aximail.33mail.com>`) with a broad
   `includeIf "gitdir:~/projects/OEC/"` layer (`~/.gitconfig-oec`) that
   overrides email to `dev@openenergy.org.au` and adds SSH commit signing —
   but never touches `user.name`, and applies identically to every repo
   under the whole OEC project tree, Gitea and GitHub alike. `ha-tariff-au`
   and `tariff-service` both had explicit local overrides to `oec-agent`
   already; this repo was the one GitHub-owned repo missing that override,
   so it silently inherited the founder's personal identity by default.

**Confirmed NOT affected:** `tariff-service` (HTTPS remote from the start,
routes through `gh`'s registered `credential.https://github.com.helper`,
which correctly resolves to whichever `gh` account is active — `oec-agent`
throughout this session, verified via the stored credential's username).
The 5 Gitea-hosted repos (`fleet-core`, `fleet-evaluator`,
`ha-fleet-connect`, `infra`, `website`) were explicitly out of scope per
founder's direction (personal `zitelli` Gitea identity there is
intentional, not a gap) — audited anyway for completeness: all commit
history in those repos is consistently `axilotl <dev@openenergy.org.au>`
end-to-end, no `oec-agent` commits and no metadata/auth mismatch, i.e. no
hidden surprise there either way.

**Fixed, this session, founder-approved at each step:**
- `ha-tariff-au` and this repo's `origin` remotes switched SSH → HTTPS
  (`https://github.com/Open-Energy-Collective/<repo>.git`), routing pushes
  through `gh`'s credential helper instead of raw SSH key resolution.
- This repo's `.gitmodules` submodule URL for `home-assistant/tariff-au`,
  and the submodule's own checked-out `origin`, switched SSH → HTTPS too
  (`git submodule sync` propagated the second one). Commit `5ed0f7e`.
- Founder ran `gh auth logout --hostname github.com` for the personal
  `axilotl` account on this machine, leaving only `oec-agent` authenticated
  — removes the credential from `gh`'s reach entirely, not just from being
  "currently inactive."
- This repo given an explicit local `git config user.name/email` override
  (`oec-agent <oec-agent@users.noreply.github.com>`), matching
  `ha-tariff-au`/`tariff-service`, so it no longer depends on global-config
  fallback behavior at all.
- Verified afterward: `git remote -v` across every repo under `repos/`
  shows zero remaining `git@github.com:` SSH remotes anywhere (top-level
  repos or submodules); every repo is 0-commits-ahead of its upstream
  (nothing outstanding left unpushed from this incident).

**Not changed, deliberately:** the mistaken commit itself (`5505f8e`,
"chore: bump ha-tariff-au submodule 95495fd -> b9e0dc8") was left as-is on
`main` — founder's explicit call, content is a harmless one-line submodule
pointer bump with no sensitive data, and rewriting published history for it
wasn't judged worth the disruption.

## 2026-08-06 — HANDOFF #1 (submodule fix) already satisfied, no action taken

`planning/tracking/TODO-fleet-core.md`'s 2026-08-06 HANDOFF #1 asked to convert
`home-assistant/tariff-au/` from "a duplicate clone" into a real git submodule,
describing it as a leftover from a previously-reverted submodule-workflow test.

Checked current repo state before doing anything: this is already a correct,
working git submodule, not a duplicate clone.

- `.gitmodules` present, correct entry: `submodule.home-assistant/tariff-au.path`
  / `.url = git@github.com:Open-Energy-Collective/ha-tariff-au.git`
- `git ls-files -s home-assistant/tariff-au` shows mode `160000` (gitlink), not a
  tracked directory of files
- `home-assistant/tariff-au/.git` is a pointer file (`gitdir:
  ../../.git/modules/home-assistant/tariff-au`), not a nested `.git` directory
- `git submodule status` clean, pinned at `a148464`
- `git status --short` clean, no untracked/duplicate files anywhere
- Most recent commit on this path is `41cfbd4` ("chore: bump ha-tariff-au
  submodule to a148464") — a routine submodule bump, which wouldn't exist if this
  were still a duplicate checkout

All 5 of the handoff's own acceptance criteria are already met. No code change
made — flagging that the handoff's premise is stale rather than actioning it, per
`repos/.agent/context.md`'s rule that a planning-doc/reality mismatch is a finding
to report, not something to silently "fix" by doing unnecessary work.

## 2026-08-10 — Security/PII scan pass: no `.gitignore` existed at all, added; steering doc added

**Cross-repo security pass** (`repos/.agent/security-scan.md`, new repeatable
process this session, precedent `ha-tariff-au` commit `b5d2119`). Working tree and
history clean — this repo is just a submodule-pointer wrapper (3 tracked files),
low surface area. Gap found: no `.gitignore` existed at all. Added the standard
secrets/agent-local-state exclusions used across the other repos. Also added a
first `CLAUDE.md`/`.agent/context.md` (this repo had neither) pointing at
`repos/.agent/context.md`'s org-wide rules, so this repo picks up the security-scan
process (and everything else) without needing it copied in.
