# Branch / worktree landing plan — dundore-homelab (2026-10-11)

Owner: PM lead. Status: Phase 0–2b complete (read-only). Phases 3–6 await execution.
Operator approved this plan verbatim on 2026-10-11 ("I agree with the plan you recommend").

## Measured basis (do not re-derive)
- `git fetch --prune origin` run with authorization; classification re-run against fresh `origin/dev`.
- 36 local branches, 27 linked worktrees. Main clone `/home/wdundore/git/dundore-homelab` is on
  `fix/tailnet-transport-identity` with one pending change (` M Vagrantfile`) that is NOT ours.
- Buckets: MERGED 3 · DUPLICATE (patch-id) 0 · SUPERSEDED (content test) 0 · ALIVE 33.
  Excluding the `prod` trunk, the cleanup universe is **32 feature/fix branches**.
- Artifacts: `/tmp/archive-20261011.bundle` (complete history, all branches+tags, `bundle verify` OK),
  `/tmp/wt-triage2-20261011.md` (36 rows: ahead/dup/uniq/class/last/author/worktree),
  `/tmp/wt-supersede-20261011.md` (33 SUPERSEDED|ALIVE verdicts with ± and touched paths).
  /tmp is volatile — re-create the bundle before any destructive step.
- 19 of 36 branches are foreign-authored (`ahab-d701-builder`, `ahab-pm`, `pm-b`, …).
- `/tmp` is wiped between sessions; the bundle is the only rollback besides archive tags.

## Important qualifier found while closing out Phase 2b
Several branch tips are titled `wip(capture): preserve uncommitted work ahead of cleanup - unreviewed`
(e.g. `idp-b1-catalog`, `fix/helpdesk-443`, `feat/fleet-secret-parity-59`) — another session snapshotted dirty
work into branches rather than finishing it. Treat those as **unreviewed captures**: they must be read and
reviewed before any landing, never rebased-and-merged on the strength of being ALIVE. A supplementary
check (`/tmp/wt-branchnewfiles-20261011.md`) lists, per branch, the files it adds and how many of those are
absent from the trunk tree — most are absent, confirming the work is genuinely unlanded rather than renamed.

## Conclusion that governs everything below
Nothing here is stale clutter. Pruning would destroy real work. The mess is 32 branches of
unlanded work; the correct cleanup is **land or explicitly shelve**, never silent delete.

## Phase 3 — reap only what is provably merged  (2 items, low risk)
1. `git -C /home/wdundore/git/dundore-homelab bundle create /tmp/archive-pre-reap-$(date +%F).bundle --branches --tags`
2. Tag, then remove: `git tag archive/jenkins-union-20261011 jenkins-union-20261011` (same for `jenkins-on-dev`),
   `git worktree remove /home/wdundore/git/wt-jenkins-union`, `git branch -D jenkins-union-20261011 jenkins-on-dev`,
   `git worktree prune`.
   Leave `production` alone: merged but foreign-authored.
Rollback: `git worktree add … archive/<name>` or reset the branch from the bundle.

## Phase 4 — collapse the topic pairs first (where duplicate effort actually lives)
One landing per cluster, not one per branch. Verify each pair by content, then land the better one
and record the other as superseded-by-<sha>.
- `fix/human-account-guard` + `fix/human-account-guard-v2`
- `fix/galaxy-role-resolution` + `fix/galaxy-roles-on-trunk`
- `fix/awx-agent-access-25` + `fix/awx-fleet-push`
- `docs/blueprint-delta`, `docs/coldstart-bootstrap`, `docs/nas-decoy-d02`, `docs/pre-refactor-audit`,
  `docs-l0-20261008`, `docs-ssot-align-20261001` → one docs-convergence landing
- `fix/helpdesk-443` + `fix/helpdesk-note-attribution-63` + `feat/helpdesk-queue` + `spike/helpdesk-untracked`
- identity cluster: `fix/git-identity-12` + `fix/makefile-ssh-identity-51` + `fix/tailnet-transport-identity`
  — blocked on the fleet identity ruling, see Open items.

## Phase 5 — land the rest, one cluster per session, gate-gated
For each cluster: rebase onto `origin/dev` → `make gate` (all five steps) → merge to `dev` → push →
`docs/audits/` row naming the branch, the ticket, the gate result, and the reviewer.
- Ticket-numbered branches map to fleet helpdesk tickets (`-59`, `-63`, `-25`, `-51`, `-12`, `-48`, `-57`);
  cross-check the ticket state in `~/git/ahab-config/queue/todo.txt` before landing — several may be closed.
- Known land-blocker: `make gate` step 4/5 cannot run locally (`ansible-lint` crash,
  "Python 3.14 requires ansible-core >= 2.20.0, found 2.18.18rc1"). Until fixed, treat lint as NOT RUN
  and say so in the audit row. `local.yml --syntax-check` also fails on missing role `chrony`
  (`roles/base/meta/main.yml:19`).
- Do not merge `prod` into `dev` or vice versa unilaterally; trunks are reconciled by operator decision.

## Phase 6 — shelve explicitly, so the next session stops re-triaging
Branches whose work is deliberately parked (`shelve/student-safety-law-20260819`, `spike/helpdesk-untracked`,
`reconcile/2026-10-03`, `feature/student-safety-law`, `feature/sre-vagrant-baseline`, `idp-b1-catalog`) get one
line each in this file: parked / reason / revisit-when. Keep the branch; remove the worktree only if the
operator names it.

## Standing guardrails
- Never delete a foreign-authored branch. Archive tag + report line only.
- Never stash, revert, checkout, or clean a dirty tree. The main clone's `Vagrantfile` carries someone
  else's pending hunks (`vm_name` removals, Fedora-43 `box_url` pin) plus our `synced_folder` line 11 —
  commit ruling still pending with the operator.
- Fresh bundle before every destructive phase; archive tag before every branch removal.
- `origin/dev` freshness: fetch is authorized for this repo; re-fetch before each landing session.
- Push transport is persisted repo-local (`core.sshCommand` = `~/.ssh/id_ed25519_automation`) on
  `dundore-homelab`; `~/.pm` uses `pm-github`. `ahab`/`ahab-config` deploy keys are READ-ONLY,
  `dundore-dnscontrol`/`scripts` reject both keys — do not attempt pushes there.

## Open items this plan depends on
1. Fleet SSH identity ruling (`keys/service_id` absent ⇒ `-e ansible_ssh_private_key_file=$HOME/.ssh/ansible_user`
   on every run; `a0d14ab` deliberately NOT ported to `dev`).
2. `Vagrantfile` commit ruling (mixed authorship in one file).
3. Write keys for `ahab-config` (queue commits `2b2a6b7`, `9d06f47`, `c0cbeaf` are local-only) and `ahab`.
4. Gate 4/5 environment fix (ansible-lint vs ansible-core) and the missing `chrony` role.
5. Make the gate a *required* push check (`.github/workflows/ci.yml` exists and triggers on push to
   `prod`/`dev`, but enforcement is unproven and CI omits `transport-check`).

## Phase 3 executed — 2026-10-12 (reap)

- bundle: /tmp/archive-pre-reap-2026-10-11.bundle, 2,001,349 bytes, `git bundle verify` exit 0 ("is okay", 40 refs); /tmp is volatile — the tags below are the real rollback.
- rollback tags: archive/jenkins-union-20261011 -> c1526c7, archive/jenkins-on-dev -> 1f01491; restore with `git branch <name> archive/<name>` then `git worktree add`.
- deleted branches: jenkins-union-20261011 (tip c1526c7, identical to origin/dev) and jenkins-on-dev (tip 1f01491, proven contained in origin/dev by `merge-base --is-ancestor` exit 0).
- removed worktrees: /home/wdundore/git/wt-jenkins-union (clean) and /home/wdundore/git/homelab-jenkins-dev (clean). DEVIATION: the second worktree is not named in the plan; its removal was required because jenkins-on-dev was checked out there, so the plan's `git branch -D` could not otherwise run. Both removals were gated on `merge-base --is-ancestor ... origin/dev` exit 0 AND a zero-line `status --porcelain`.
- post-state: 34 local branches (was 36), 25 worktree entries (was 27), zero jenkins branches or worktrees, origin/dev still c1526c7, origin/prod still 0333656. `production` untouched (foreign-authored).
- independent audit verdict: PASS on the reap; expected-vs-actual delta empty on 7 of 8 checks. The 8th check flagged that the main checkout's uncommitted `M Vagrantfile` had become commit 37613b8 "test" (author and committer ahab-d701-builder, 2026-10-12T00:26:06Z) on branch fix/tailnet-transport-identity, 4m17s AFTER the reap's last log write (00:21:49Z) — i.e. third-party work, not a reap side effect. Content: D-79 synced_folder/rsync transport block, fed.vm.box_url amd64 pin, removal of two lib.vm_name lines. Left untouched; still pending the operator's Vagrantfile ruling.
