---
name: resolving-git-conflicts
description: Diagnose and resolve git conflicts of any kind — merge, rebase, cherry-pick, stash, revert. Use this skill EVERY time conflicts appear during work, or whenever the user mentions merge conflicts, rebase conflicts, conflict markers, "both modified" files, a failed or conflicted git pull, or asks to "fix conflicts". Use it even when the resolution looks obvious — many conflicts are phantom artifacts of squash merges or rewritten upstream history, and the correct fix is a different git strategy (e.g. git rebase --onto), not editing conflict markers.
---

# Resolving Git Conflicts

Core rule: **diagnose before you resolve.** A conflict is a symptom, not the problem. Editing conflict markers is the right treatment only for genuinely divergent edits. A large class of conflicts — especially in stacked-branch workflows — are *phantom conflicts* created by squash merges, rebased or force-pushed upstreams, or otherwise duplicated commits. Hand-resolving those re-applies work that already landed, pollutes history with bogus merge commits, and frequently reintroduces stale code. When the cause is structural, the right fix is to abort and re-approach with a different strategy, after which most conflicts simply don't exist.

Work through the phases in order. Phases 1–2 are read-only and cheap; they determine everything else.

**Autonomy policy:** act without asking for confirmation, including aborting the in-progress operation and rewriting *local* history — the safety snapshot plus the reflog make this fully recoverable. The single exception: if the fix rewrites commits that were already pushed, finish the work locally, then **stop and ask the user before any force-push**, and use `git push --force-with-lease` (never bare `--force`).

## Phase 0 — Safety snapshot

Before changing anything, record where things stand so any outcome is recoverable:

```bash
git status                          # full picture of the conflicted state
git stash list                      # note pre-existing stashes; never clobber them
# Back up the pre-operation branch tip:
git branch conflict-backup-$(date +%s) ORIG_HEAD 2>/dev/null \
  || git branch conflict-backup-$(date +%s) HEAD
```

During a merge, `HEAD` is still the pre-merge branch tip; during a rebase, the pre-rebase tip lives in `ORIG_HEAD` (HEAD is detached mid-replay). Don't ask permission for the backup branch — it's free. Mention it in the final report and offer to delete it.

If the working tree had uncommitted changes *unrelated* to the operation (git refused to start, or they're mixed in), deal with that first — usually `git stash push` before strategy changes, `git stash pop` after.

## Phase 1 — Identify the operation

`git status` already names the operation, but verify against the state files — status text varies across git versions and the state files also give you the refs involved:

| Check | Meaning |
|---|---|
| `.git/MERGE_HEAD` exists | merge in progress (includes a conflicted `git pull` with merge) |
| `.git/rebase-merge/` or `.git/rebase-apply/` exists | rebase in progress (includes `git pull --rebase`) |
| `.git/CHERRY_PICK_HEAD` exists | cherry-pick in progress |
| `.git/REVERT_HEAD` exists | revert in progress |
| none of the above, but `git status` shows unmerged paths | conflicted `git stash apply/pop` |

Then gather the actors:

```bash
# merge:
git log -1 --oneline MERGE_HEAD            # what is being merged in
# rebase:
cat .git/rebase-merge/head-name            # branch being rebased (refs/heads/...)
cat .git/rebase-merge/onto                 # commit being rebased onto
git log -1 --oneline REBASE_HEAD           # the commit currently being replayed
# any operation:
git log -1 --oneline ORIG_HEAD             # tip before the operation started
```

**Critical: `ours`/`theirs` semantics invert under rebase.** In a merge, `ours` = your branch, `theirs` = what's coming in. A rebase replays *your* commits onto the target, so `ours` = the new base (e.g. main) and `theirs` = *your own commit*. Getting this backwards silently destroys work. Before trusting the labels, confirm with `git log -1 REBASE_HEAD` what "theirs" actually is.

## Phase 2 — Map the topology and find the cause

Still read-only. Establish the shape of history:

```bash
git diff --name-only --diff-filter=U                 # the conflicted files
incoming=$(git rev-parse MERGE_HEAD 2>/dev/null || cat .git/rebase-merge/onto)
base=$(git merge-base HEAD "$incoming")
git log --oneline --graph HEAD "$incoming" -30       # visualize both sides
git log --oneline --left-right HEAD..."$incoming"    # commits unique to each side
```

Then ask the key question for each conflicted file: **did this branch's own commits ever touch it?**

Careful with the range: `$base` above is the fork point from the *target* branch, so on a stacked branch `$base..HEAD` also contains the parent branch's inherited commits — checking against it masks the signal. First establish where this branch's own work actually starts:

```bash
git reflog show <branch> | tail -3      # the "branch: Created from ..." entry
git log --oneline --graph HEAD          # parent feature refs / foreign-looking commits below yours
```

If the branch was created from another feature branch (not from the target), `<own-start>` is that parent's tip — the parent ref itself if it still exists, otherwise the boundary commit where authors/subjects stop being this branch's work. Then:

```bash
git log --oneline <own-start>..HEAD -- <conflicted-file>
```

Empty output for a conflicted file is a strong **phantom-conflict signal**: the "ours" side of that conflict came from the parent branch's history, not from work done here.

Phantom-conflict checklist — any two of these together make it near-certain:

1. Conflicted files that the branch's own commits never modified (check above).
2. The branch is stacked on another feature branch rather than on the target (found while locating `<own-start>`).
3. The incoming side contains a **squash commit of that parent branch**: a single commit whose message references the parent branch or its PR, touching largely the same paths. Confirm by diff comparison — patch-ids won't match a squash, so compare cumulative diffs instead:
   ```bash
   git diff <squash-commit>^ <squash-commit> > /tmp/squash.diff
   git diff "$base" <parent-branch-tip>      > /tmp/parent.diff
   diff /tmp/squash.diff /tmp/parent.diff    # near-identical → it's the squash of the parent
   ```
4. For *rebased* (not squashed) duplicates, `git log --oneline --left-right --cherry-pick HEAD..."$incoming"` hides commits with equal patch-ids; if many of "your" commits disappear from the listing, they already landed upstream in rewritten form.
5. The upstream base itself was force-pushed: `git reflog show origin/<base-branch>` shows a non-fast-forward update.

While here, also note other structural causes: the same conflicts recurring on every rebase of a long-lived branch (→ rerere, Case C), or "deleted by us/them" entries in `git status` (→ Case E).

## Phase 3 — Pick the strategy

### Case A — stacked branch whose base was squash-merged (the classic trap)

Topology: `main ← feature-1 ← feature-2` (feature-2 stacked on feature-1). feature-1 gets squash-merged into main; pulling main into feature-2 now conflicts — often in files feature-2 never touched — because git sees feature-1's original commits and the squash commit as unrelated changes to the same lines.

Do **not** resolve these by hand. Abort and transplant only the commits that truly belong to this branch:

```bash
git merge --abort        # or: git rebase --abort, matching Phase 1's finding
git rebase --onto <new-base> <old-base> <branch>
# e.g. git rebase --onto origin/main dev1/feature-branch-1 dev2/feature-branch-2
```

`<old-base>` must be the commit *inside this branch's history* after which the branch's own work begins. A parent branch ref, if one exists, is only a hint for finding it — remotes delete branches after merging, and local copies go stale. Candidates, best first:

1. A parent ref exists (local or `origin/...`): candidate = `git merge-base <parent-ref> <branch>` — not the ref itself, because the parent may have gained commits after this branch was stacked on it.
2. No usable ref, but the branch was created on this machine: `git reflog show <branch> | tail -3` — the "branch: Created from ..." entry. Reflogs are local-only; a fresh clone won't have this.
3. No ref, no reflog: read the boundary out of `git log --oneline <new-base>..<branch>` (oldest at the bottom: the parent's commits first, then this branch's own). The switch in author and subject marks it. Cross-reference the squash commit on the target: squash-merge messages typically list the squashed commit subjects, and `git show --stat <squash-commit>` lists its paths — commits matching either belong to the parent.
4. Equivalent shortcut once this branch's own commits are counted (N): use `HEAD~N` as the old base.

**Validate the candidate before rebasing, whatever its source:**

- `git merge-base --is-ancestor <old-base> <branch>` must pass.
- `git log --oneline <old-base>..<branch>` must list **exactly this branch's own commits**: nothing authored by others or named in the squash commit's message (that means the candidate sits too low — e.g. a stale local ref from before the parent's last commits), and none of this branch's own work missing (a candidate that sits too high silently drops commits — the worst outcome).

If validation fails, walk the boundary by hand per step 3. When genuinely torn between adjacent candidates, prefer the lower one: too low merely replays a few already-merged patches (they conflict or come up empty and can be skipped after confirming they're upstream), while too high loses work.

Pitfall: `git merge-base <new-base> <branch>` is **not** the old base here — it returns the original fork point from main, *below* the parent's commits, and would replay the parent's work again.

After the rebase, expect zero conflicts. Any that do remain are real — handle via Phase 4 inside the rebase. Verify per Phase 5.

### Case B — upstream base was rebased / force-pushed

Same shape, different source of truth for the old base: `git rebase --onto origin/<base> 'origin/<base>@{1}' <branch>` (the remote-tracking reflog entry from before the fetch that brought the rewrite). If the reflog is gone, fall back to Case A's boundary inspection.

### Case C — the same conflicts keep recurring

Long-lived branch, repeatedly rebased or repeatedly merged: enable reuse of recorded resolutions before resolving this time, so it's the last time:

```bash
git config rerere.enabled true
```

Then resolve once via Phase 4. Mention to the user that future identical conflicts will auto-resolve.

### Case D — genuine divergence

Both sides really did edit the same lines with different intent. Stay inside the current operation and go to Phase 4. Aborting buys nothing here.

### Case E — modify/delete and rename conflicts

`git status` shows "deleted by us" / "deleted by them". Determine intent before choosing: was the file truly deleted, or renamed (content moved)?

```bash
git log --oneline --follow --diff-filter=ADR <deleting-side> -- <file>
```

If renamed: apply the other side's edits to the file at its *new* path, then `git rm <old-path>`. If truly deleted: decide whether the other side's edits still matter anywhere; usually honor the deletion and port any still-relevant logic to wherever it moved.

## Phase 4 — Resolve real content conflicts by intent

For each file in `git diff --name-only --diff-filter=U`:

1. **Get the base version into the markers** — the merge base shows what both sides started from, which is what makes intent legible:
   ```bash
   git checkout --conflict=diff3 -- <file>
   ```
   Now each conflict shows `ours / ||||||| base / theirs`. (Remember the rebase inversion from Phase 1.)
2. **Read each side's intent, not just its text:**
   - merge: `git log --merge -p -- <file>` (only the commits from both sides touching this file)
   - rebase: `git show REBASE_HEAD -- <file>` (the change being replayed) vs. `git log -3 -p <onto> -- <file>` (what it's landing on)
3. **Write the version that satisfies both intents.** Usually that means the incoming side's structure with this side's semantic change re-applied to it. Mechanical "keep both blocks" is almost always wrong for code. If the two intents are truly incompatible (two competing implementations of the same thing), pick the one consistent with the project's direction (newer, referenced by other new code), and clearly tell the user what was chosen and why — only ask first if the choice is genuinely 50/50 and consequential.
4. **Verify the file**, then stage it: no leftover markers (`git diff --check`), syntax-check or run fast tests if the project has them, then `git add <file>`.
5. **Continue the operation:** `git merge --continue`, `git rebase --continue`, or `git cherry-pick --continue` as appropriate. A rebase may stop again on the next commit — repeat from step 1 each time. Use `git rebase --skip` only when the stopped commit is genuinely already upstream (git usually says the patch is empty; confirm before skipping).

## Phase 5 — Verify the result

- `git status` is clean and no markers survive anywhere: `grep -rn "^<<<<<<<" --exclude-dir=.git . || true`
- History has the expected shape: `git log --oneline --graph -15`. For Cases A/B specifically, `git log --oneline <new-base>..<branch>` must list **only this branch's own commits** — no duplicates of already-merged work, no stray merge commit.
- Content sanity: `git diff <new-base>...<branch>` shows only this branch's intended changes; nothing from the parent branch re-appears, nothing of ours was lost (compare against the Phase 0 backup if in doubt: `git diff conflict-backup-... <branch> -- <own-files>`).
- Build / run tests if available.
- **Report to the user:** what caused the conflicts (in one or two sentences), which strategy was used and why, what was resolved by hand and how. If local history was rewritten and the branch exists on a remote, explain that pushing requires `git push --force-with-lease` and **ask before pushing**. Offer to delete the backup branch.

## Worked example — the squash-merge trap end to end

```text
Situation: main ← dev1/feature-branch-1 ← dev2/feature-branch-2.
dev1 squash-merged feature-branch-1 into main. dev2 ran `git pull` on main
into feature-branch-2 and got conflicts in lib.py — a file dev2 never edited.

Phase 1: .git/MERGE_HEAD exists → merge in progress; MERGE_HEAD = origin/main.
Phase 2: reflog shows the branch was created from dev1/feature-branch-1 (stacked).
         git log dev1/feature-branch-1..HEAD -- lib.py → empty: dev2 never touched it.
         origin/main has one commit "Add retry logic (#42)" whose diff equals
         git diff $base dev1/feature-branch-1 → it's the squash. Case A.
Phase 3: git merge --abort
         git rebase --onto origin/main dev1/feature-branch-1 dev2/feature-branch-2
         → replays only dev2's commits; no conflicts.
Phase 5: git log origin/main..HEAD → exactly dev2's commits. Branch was already
         pushed → tell dev2 the next push needs --force-with-lease, ask first.
```
