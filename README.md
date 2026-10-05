# git-cheatsheet

A short list of the git and GitHub CLI commands I use most.

## Everyday git

```bash
git status                  # what changed
git switch -c my-branch     # create and switch to a branch
git add -p                  # stage changes interactively
git commit -m "message"     # commit staged changes
git commit --amend          # edit the last commit
git pull --rebase           # update without a merge commit
git log --oneline --graph   # compact history
```

## Undo things

```bash
git restore file.txt            # discard unstaged changes
git restore --staged file.txt   # unstage a file
git reset --soft HEAD~1         # undo last commit, keep changes staged
git revert <commit>             # new commit that undoes a commit
git reflog                      # find lost commits
```

## Stash

```bash
git stash push -m "wip"   # save work in progress
git stash list
git stash pop             # restore and drop the latest stash
```

## GitHub CLI

```bash
gh repo clone owner/repo
gh pr create --fill
gh pr checkout 123
gh pr merge --squash
gh issue list
```
