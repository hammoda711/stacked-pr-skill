# stacked-pr

A Claude Code skill for building, rebasing and landing a stack of dependent pull requests.

A stack is a chain of branches where each one is cut from the one below it, not from `dev`. Each PR targets the branch below it, so each PR holds only its own diff and stays small enough to review.

```
dev
 └── feat/a      PR 1 → dev
      └── feat/b      PR 2 → feat/a
           └── feat/c     PR 3 → feat/b
```

The one rule: the stack moves bottom-up. Never merge a layer before the one below it. Never cut a stacked branch from `dev`.

## Install

```
/plugin marketplace add hammoda711/claude-skill-stacked-pr
/plugin install stacked-pr@claude-skill-stacked-pr
```

## Use

Run `/stacked-pr`. The skill also triggers on "stack a pr on this one", "new pr on top of this pr", "restack", "update the stack", "merge the stack", "land the stack", and "my pr depends on another pr".

## What it does

Every mode starts by reading the real stack from the PR target branches, never by guessing it. Then one of three modes:

**Mode A — stack.** A new PR on top of the one you are on. Cuts the branch from the current branch, creates the PR targeting it, and notes the dependency in the body.

**Mode B — restack.** A lower branch changed, so everything above it is stale. Rebases each layer with `git rebase --onto`, bottom-up, and force-pushes with lease.

**Mode C — land.** The stack is approved. Completes the bottom PR, retargets the next PR to `dev`, restacks the rest onto `dev`, checks the resulting diff holds only that one layer, deletes the merged branch, and repeats up the stack.

Conflicts stop the run and hand the files back to you.

## Assumes

- **Azure DevOps as the default remote.** The PR commands in the skill are `az repos pr`. Azure has no stack view, which is why the skill reads the stack from each PR's target branch.
- **GitHub works too.** The skill has a section for it: the git half is identical and only the PR commands change to `gh pr create --base`, `gh pr edit --base`, `gh pr merge`. It checks `git remote -v` to tell which it is. On GitHub, deleting a merged base branch retargets the child PR automatically, so one step is often already done.
- **A `dev` integration branch.** The bottom PR targets `dev`. A repo that integrates into `main` needs the branch name swapped.
- **The `az` CLI or the `gh` CLI, authenticated**, for whichever remote you are on.
- **Force-push allowed on your feature branches.** Restacking rewrites them. The skill uses `--force-with-lease`.

## License

MIT. See [LICENSE](LICENSE).
