---
name: stacked-pr
description: >
  Build and land a stack of dependent pull requests. Use when the user wants a
  new PR that sits on top of the PR they are already working on, when a lower
  branch in a stack changed and everything above it needs rebasing, or when the
  stack is approved and has to be merged down into dev one layer at a time.
  Reads the stack from the PR target branches, rebases with --onto, retargets
  each PR, and completes them in order. Triggers on "stacked pr",
  "/stacked-pr", "stack a pr on this one", "new pr on top of this pr",
  "restack", "update the stack", "merge the stack", "land the stack",
  "my pr depends on another pr".
---

# Stacked PRs

A stack is a chain of branches where each one is cut from the one below it, not
from `dev`. Each PR targets the branch below it. Only the bottom PR targets
`dev`.

```
dev
 └── feat/a      PR 1 → dev
      └── feat/b      PR 2 → feat/a
           └── feat/c     PR 3 → feat/b
```

Each PR holds only its own diff, so each one is small enough to review.

**The one rule: the stack moves bottom-up.** Never merge a layer before the one
below it. Never cut a stacked branch from `dev`.

This skill has three modes. Read the user's ask and pick one:

| The user wants | Mode |
|---|---|
| A new PR on top of the current one | **A — Stack** |
| A lower branch changed, fix the ones above | **B — Restack** |
| The stack is approved, get it into dev | **C — Land** |

---

## Phase 0 — read the stack (every mode starts here)

Never guess the stack. Azure DevOps has no stack view, so the PR target
branches *are* the stack. Read them.

```bash
git fetch origin --prune
git status --short
git branch --show-current
az repos pr list --status active \
  --query "[].{id:pullRequestId,src:sourceRefName,tgt:targetRefName,draft:isDraft,merge:mergeStatus}" \
  -o table
```

Strip `refs/heads/` from both columns. Start at the current branch and follow
`tgt` down until you reach `dev`. That chain is the stack. Walk `src` upward the
same way to find the layers above.

Show the user the stack as a tree with the PR id on each layer, plus `dev` at
the bottom, before doing anything.

**Stop and report if:**

- The working tree is dirty. Name the files. The user decides.
- Two PRs have the same source branch. The stack is ambiguous.
- A PR targets a branch that no longer exists on the remote. That layer is
  orphaned — a lower branch was deleted too early. Say which, and offer mode C
  step 3 to repair it.
- The current branch has no PR at all. Then there is no stack yet; mode A still
  works and the new PR targets the current branch.

---

## Mode A — stack a new PR on the current one

Use when the user is on a branch with an open PR and wants the next piece of
work to build on it.

**1. Check the branch is pushed.** A stacked PR cannot target a branch the
remote does not have.

```bash
git ls-remote --heads origin "$(git branch --show-current)"
git log --oneline "origin/$(git branch --show-current)..HEAD"
```

If the second command prints commits, the parent branch is behind the remote.
Say so and ask before pushing it.

**2. Cut the new branch from the current one, not from dev.**

```bash
git checkout --no-track -b <new-branch>
```

`--no-track` is not optional. Without it a bare `git push` can go somewhere the
user did not intend.

Follow the repo's branch naming. Read it, do not invent it:

```bash
git branch -a --sort=-committerdate | head -20
```

**3. The user does the work and commits.** This skill does not write code.

**4. Push the branch.** Ask first — pushing is the user's call.

```bash
git push -u origin HEAD:refs/heads/<new-branch>
```

**5. Open the PR against the parent branch, not dev.**

```bash
az repos pr create \
  --source-branch <new-branch> \
  --target-branch <parent-branch> \
  --title "<title>" \
  --description "Part N of the <feature> stack." "" "Depends on !<parent-pr-id>. Review and merge that one first." \
  --repository <repo> \
  --squash false
```

`--target-branch <parent-branch>` is the whole point. Target `dev` and the PR
shows the parent's diff too, and the reviewer sees work they already approved.

Always put the dependency in the description. Azure DevOps shows no stack, so
the description is the only place a reviewer learns the order.

**Do not pass `--delete-source-branch`.** Deleting a branch that a layer above
still targets orphans that PR. Branches get deleted in mode C, last.

---

## Mode B — restack after a lower branch changed

Use when the user pushed a fix to a lower layer (usually from review feedback)
and the layers above still carry the old commits.

Git 2.38+ does the whole stack in one rebase. Check first:

```bash
git --version
```

**1. From the top branch of the stack:**

```bash
git checkout <top-branch>
git rebase --update-refs <bottom-branch>
```

`--update-refs` replays every commit from the bottom up and moves each
intermediate branch pointer to its rewritten commit. Without it only the top
branch moves and the middle layers are left pointing at commits that no longer
exist.

A branch checked out in another git worktree will **not** move. Check
`git worktree list` first and switch it away.

**2. Push every branch that moved.** Ask before pushing.

```bash
git push --force-with-lease origin <branch-1> <branch-2> <branch-3>
```

Never `--force`. `--force-with-lease` refuses when the remote has commits you
have not seen, which is what stops you from erasing a teammate's push.

**3. Tell the user what votes they just lost.** If the repo resets votes on new
pushes, every layer you pushed needs re-approval even though its code did not
change. That is a real cost of a restack; name it rather than letting them find
out.

---

## Mode C — land the stack into dev

One layer at a time, bottom first. The order inside each layer matters more
than the commands.

**Before anything, save the bottom branch tip.** You need it after the merge and
you cannot get it back once the branch is gone.

```bash
OLD_PARENT=$(git rev-parse origin/<bottom-branch>)
echo "$OLD_PARENT"
```

Then, for each layer from the bottom:

### Step 1 — complete the bottom PR

**Ask the user first. Every time.** Completing a PR is a merge into `dev`; it is
never this skill's decision. Show the PR id, title, and `mergeStatus`, then wait.

```bash
az repos pr update --id <bottom-pr-id> --status completed
```

Add `--squash true` only if the user asks for it. This repo merges with a merge
commit by default, which keeps the layer's own commits in `dev`.

**Never pass `--delete-source-branch` here.** The layer above still targets this
branch.

### Step 2 — retarget the next PR to dev

`az repos pr update` has no `--target-branch` flag. The REST API does support
it, so go through `az devops invoke`:

```bash
echo '{"targetRefName": "refs/heads/dev"}' > /tmp/retarget.json

az devops invoke \
  --area git --resource pullrequests \
  --route-parameters project=<project> repositoryId=<repo> pullRequestId=<next-pr-id> \
  --http-method PATCH --api-version 7.1 \
  --in-file /tmp/retarget.json --encoding utf-8 \
  --query "{id:pullRequestId,tgt:targetRefName}" -o json
```

Check the output says `refs/heads/dev`. If the call is rejected, PR retargeting
is off for the organisation; fall back to the web UI, **More actions → Change
target branch**, and say so rather than carrying on.

### Step 3 — restack the rest onto dev

This is the step people get wrong. A plain `git rebase origin/dev` replays the
bottom layer's commits a second time, because the merge made new commit ids for
them. Use `--onto` so only the layer's own commits move:

```bash
git fetch origin
git checkout <top-branch>
git rebase --update-refs --onto origin/dev "$OLD_PARENT" <top-branch>
```

Read it as: take the commits after `$OLD_PARENT` up to `<top-branch>`, and
replay them on `origin/dev`. `--update-refs` carries every branch in between.

This is correct whether the PR was merged with a merge commit, a squash, or a
rebase, so use it in all cases.

Push every branch that moved:

```bash
git push --force-with-lease origin <branch-2> <branch-3>
```

### Step 4 — check the diff is only this layer

Prove it before telling the user it worked:

```bash
git diff --stat origin/dev...<next-branch>
az repos pr show --id <next-pr-id> --query "{tgt:targetRefName,merge:mergeStatus}" -o json
```

The diff must show only the files that layer touches. If the lower layer's files
are in there, the `--onto` base was wrong. Stop and re-read the stack.

### Step 5 — now delete the merged branch

Only once the layer above is retargeted and pushed:

```bash
git push origin --delete <bottom-branch>
git branch -D <bottom-branch>
```

### Step 6 — repeat

The next layer is now the bottom. Set `OLD_PARENT` to its old tip and go back to
step 1. Keep going until the last PR targets `dev` and merges.

---

## Conflicts

A conflict can stop any rebase. It is normal in a stack: the layer below moved.

```bash
# resolve the files, then
git add .
git rebase --continue
```

To back out completely:

```bash
git rebase --abort
```

`--abort` puts every branch in the stack back where it was, including the ones
`--update-refs` already moved. It is always safe to use.

If the same conflict comes back on every layer, the split is wrong — two layers
are editing the same lines. Say so; the fix is to reorder or merge the layers,
not to keep resolving.

---

## Safety rules

These hold in every mode.

1. **Ask before pushing, merging, or completing a PR.** Branching, rebasing and
   reading are fine without asking.
2. **`--force-with-lease`, never `--force`.**
3. **Delete a branch only after the layer above it has been retargeted.**
4. **One person owns a stack.** A rebase rewrites history on every branch in it.
   If someone else has pulled a layer, stop and tell the user before rewriting.
5. **Keep the stack short.** Three or four layers. Each extra layer is another
   full restack after every merge.
6. **Never rebase `dev`.**

---

## If the remote is GitHub, not Azure DevOps

Check with `git remote -v`. The git half is identical. Only the PR commands change:

```bash
gh pr create --base <parent-branch> --head <new-branch> --title "..." --body "Depends on #<parent-pr>"
gh pr list --state open --json number,headRefName,baseRefName
gh pr edit <next-pr> --base dev
gh pr merge <bottom-pr> --merge
```

GitHub retargets a child PR by itself when the base branch is deleted, so step 2
is often done for you. Verify it with `gh pr view <next-pr> --json baseRefName`
instead of assuming.

---

## Report at the end

Short, in this order:

1. The stack as a tree, with PR ids, as it stands now
2. What changed: branches created, rebased, pushed, PRs created, retargeted or
   completed
3. The verification you actually ran — the `git diff --stat` output and the PR
   target branch, quoted
4. What is left to do, and anything you skipped or could not do

Never say a layer landed without having read its PR status back from the API.
