# Advanced GIT course

## Topics Covered

- Conflicts with rebase and merge
- Our work vs. their work
- Git head, reflog, and recovering work
- Git stash and worktrees
- Git bisect
- Git revert and cherry-pick
- Tags and releases

## Forking the Repo of MegaCorp

Create an own of the megacorp starter repo! 

A [fork](https://docs.github.com/en/get-started/quickstart/fork-a-repo) is a copy of a repository. Forking a repository allows you to freely experiment with changes without affecting the original project.

Forking _is not_ a Git operation, but it is a feature offered by many Git hosting services such as [GitHub](https://github.com), [GitLab](https://gitlab.com), and [Bitbucket](https://bitbucket.org).

## Clone MegaCorp

Clone down my fork of the Megacorp repository.

```sh
git clone https://github.com/Roshani-Udupa/megacorp.git
```

Check if you are in main branch with the following command

```sh
git branch
```

## HEAD

Branches are references to commits, and `HEAD` is a reference to the branch you're currently on.

```sh
cat .git/HEAD 
```

Output:
```sh
ref: refs/heads/main
```

## Reflog Command

The [`git reflog` command](https://git-scm.com/docs/git-reflog)  is kinda like `git log` but stands for "reference log" and specifically logs the changes to a "reference" that have happened over time.

Reflog uses a different format to show the history of a branch or HEAD: one that's more concerned with the number of steps back in time.

`git reflog` keeps track of where all of your references have been.

## Merge Command

Using Git internals is exceptionally inconvenient. We had to copy/paste and use the `cat-file` command 3 times!

I would not recommend doing it that way, but I wanted to drive home the point that you can always drop down to the plumbing commands if needed.

_Luckily, there is a better way._ The [`git merge`](https://git-scm.com/docs/git-merge) command actually takes a "commitish" as an argument:

```sh
git merge <commitish>
```

A "commitish" is something that _looks_ like a commit (branch, tag, commit, HEAD@{1})

In other words instead of:

```sh
git reflog # find the commit sha at HEAD@{1}
git cat-file -p <commit sha>
git cat-file -p <tree sha>
git cat-file -p <blob sha> > slander.md
git add .
git commit -m "B: recovery"
```

We could have just done:

```sh
git merge HEAD@{1}
```


## Conflicting Changes

### Merge Conflict
A merge conflict occurs when two commits _modify the same line_ and Git can't automatically decide which change to keep and which change to discard.
Consider the following commit history:

```bash
    C     feature
  /
A - B     main
```

The `main` branch has a file with these lines:

```go
package main

func isNice(num int) bool {
    return num == 69 // commit B changed this line
}
```

While `feature` has:

```go
package main

func isNice(num int) bool {
    return num == 420 // commit C changed this line
}
```

If we merge `feature` into `main`, Git will detect that the `return` line was changed in both branches independently: which creates a [conflict](https://git-scm.com/docs/git-merge#_how_to_resolve_conflicts).

When a conflict happens (usually as the result of a `merge` or `rebase`) Git will prompt you to _manually_ decide which change to keep. It's okay when the same line is modified in one commit, and then again in a _later_ commit. The problem arises when the same line is modified in two commits that aren't in a parent-child relationship.

## RESET

```sh
git reset --hard <commit-hash>
```

### Checkout Conflict

The [`git checkout` command](https://www.git-scm.com/docs/git-checkout) can checkout the individual changes during a merge conflict using the `--theirs` or `--ours` flags.

- `--ours` will overwrite the file with the changes from the branch you are currently on and merging into
- `--theirs` will overwrite the file with the changes from the branch you are merging into the current branch

```bash
git checkout --theirs path/to/file
```


### Rebase conflicts

In the "real world," what happens most often is:

1. You switch to a new branch, say `fix_bug`, which is a copy of `main`.
2. While you're fixing the bug, someone else merges _their_ changes into `main`.
3. You fix the bug, and it so happens that you edited the same files (and lines) that the other person did.
4. You open a Pull Request to merge (or rebase) `fix_bug` into `main`, then Git tells you there's a conflict.
5. You resolve the conflict on your branch.
6. You complete the Pull Request with the conflict resolved.

Note when you rebase a branch - and there is a conflict then,
- If you had merged `main` instead of using `rebase`, `HEAD` would still point to `banned` because git doesn't switch branches under-the-hood during a merge. However, you kinda need to think in reverse with `rebase`... `--theirs` represents the `banned` branch which, in reality, is usually "our" changes.
- Run `git branch`. Notice that you're _not on a branch!_. You should see something like this:

```
* (no branch, rebasing banned)
  banned
  main
```

Because you're in the middle of a rebase, you're in a special state called "detached HEAD." This is a temporary state that allows you to resolve the conflict before you continue the rebase.

> Note: "Accept incoming change" is the same as `git checkout --theirs`, and "Accept current change" is the same as `git checkout --ours`.

> In a merge, `--ours` refers to your current branch, but in a rebase, `--ours` refers to the branch you're rebasing _onto_.

> With `rebase` conflicts, unlike merge conflicts, we don't `commit` to resolve the conflict. Instead, we `--continue` the rebase.

# Repeat Resolution Setup

A common complaint about rebase is that there are times when you may have to manually resolve the same conflicts over and over again. This is especially prevalent when you have a long-running feature branch, or even more likely, multiple feature branches that are being rebased onto `main`.

## RERERE to the Rescue

> The [git rerere](https://git-scm.com/book/en/v2/Git-Tools-Rerere) functionality is a bit of a hidden feature. The name stands for “reuse recorded resolution” and, as the name implies, it allows you to ask Git to remember how you’ve resolved a hunk conflict so that the next time it sees the same conflict, Git can resolve it for you automatically.

In other words, if enabled, `rerere` will remember how you resolved a conflict (applies to rebasing but also merging) and will automatically apply that resolution the next time it sees the same conflict. 

## Squashing

1. Start an interactive rebase with the command `git rebase -i HEAD~n`, where `n` is the number of commits you want to squash.
2. Git will open your default editor with a list of commits. Change the word `pick` to `squash` for all but the first commit.
3. Save and close the editor.

The `-i` flag stands for "interactive," and it allows us to edit the commit history before Git applies the changes. `HEAD~n` is how we reference the last `n` commits. `HEAD` points to the current commit (as long as we're in a clean state) and `~n` means "n commits before HEAD."

## Rename a branch

```sh
git branch -m new-branch-name
```

## Force Push

So we did some naughty stuff. We squashed `main` which means that because our _remote_ `main` branch on GitHub has commits that we removed, `git` won't allow us to push our changes because they're totally out of sync.

The [`git push`](https://git-scm.com/docs/git-push) command has a `--force` flag that allows us to overwrite the remote branch with our local branch. It's a very dangerous (but useful) flag that overrides the safeguards and just says "make that branch the same as this branch."

```bash
git push origin main --force
```

## Git Stash

The [`git stash` command](https://git-scm.com/docs/git-stash) records the current state of your working directory _and the index_ (staging area). It's kinda like your computer's copy/paste clipboard. It records those changes in a safe place and reverts the working directory to match the `HEAD` commit (the last commit on your current branch).

To stash your current changes and revert the working directory to match `HEAD`:

```sh
git stash
```

To list your stashes:

```bash
git stash list
```
Stash has [a few options](https://git-scm.com/docs/git-stash), but the ones that you will use most are:

```sh
git stash
git stash pop
git stash list
```

The `pop` command will (by default) apply your _most recent_ stash entry to your working directory and remove it from the stash list. It effectively undoes the `git stash` command. It gets you back to where you were.

### Apply Without Removing from Stash

This will apply the most recent stash changes, just like `pop`, but it will keep the stash in the stash list.

```sh
git stash apply
```

### Remove a Stash Without Applying

This will remove the most recent stash from the stash list without applying it to your working directory.

```sh
git stash drop
```

## Revert

Where [`git reset`](https://git-scm.com/docs/git-reset) is a sledgehammer, [`git revert`](https://git-scm.com/docs/git-revert) is a scalpel.

A revert is effectively an _anti_ commit. It does not _remove_ the commit (like `reset`), but instead creates a new commit that does the exact opposite of the commit being reverted. It undoes the change but keeps a full history of the change and its undoing.

### Using Revert

To revert a commit, you need to know the commit hash of the commit you want to revert. You can find this hash using `git log`.

```sh
git log
```

Once you have the hash, you can revert the commit using `git revert`.

```sh
git revert <commit-hash>
```

## Cherry Pick

There comes a time in every developer's life when you want to yoink a commit from a branch, but you don't want to merge or rebase because you don't want _all_ the commits.

The [`git cherry-pick` command](https://git-scm.com/docs/git-cherry-pick) solves this.

```sh
git cherry-pick <commit-hash>
```


## Git Bisect

> How do we find out when a bug was introduced?

That's where the [`git bisect` command](https://git-scm.com/docs/git-bisect) comes in. Instead of manually checking _all_ the commits (`O(n)` for Big O nerds), `git bisect` allows us to do a binary search (`O(log n)` for Big O nerds) to find the commit that introduced the bug.

For example, if you have 100 commits that _might_ contain the bug, with `git bisect` you only need to check 7 commits to find the one that introduced the bug.

>`git bisect` isn't **just** for bugs, it can be used to find the commit that introduced **any** change, but issues like bugs and performance regressions are a common use case.

### There are effectively 7 steps to bisecting:

1. Start the bisect with `git bisect start`
2. Select a "good" commit with `git bisect good <commitish>` (a commit where you're sure the bug wasn't present)
3. Select a bad commit via `git bisect bad <commitish>` (a commit where you're sure the bug was present)
4. Git will checkout a commit between the good and bad commits for you to test to see if the bug is present
5. Execute `git bisect good` or `git bisect bad` to say the current commit is good or bad
6. Loop back to step 4 (until `git bisect` completes)
7. Exit the bisect mode with `git bisect reset`

## What Is a Worktree?

A worktree (or "working tree" or "working directory") is just the directory on your filesystem where the code you're tracking with Git lives. Usually, it's just the root of your Git repo (where the `.git` directory is). It contains:

- Tracked files (files that Git knows about)
- Untracked files (files that Git doesn't know about)
- Modified files (files that Git knows about that have been changed since the last commit)

## The Worktree Command

Git has the [`git worktree` command](https://git-scm.com/docs/git-worktree) that allows us to work with worktrees. The first subcommand we'll worry about is simple:

```sh
git worktree list
```

It lists all the worktrees you created.

## The Worktree Command

Git has the [`git worktree` command](https://git-scm.com/docs/git-worktree) that allows us to work with worktrees. The first subcommand we'll worry about is simple:

```sh
git worktree list
```

It lists all the worktrees you created.

### Linked Worktrees

We've talked about:

1. Stash (temporary storage for changes)
2. Branches (parallel lines of development)
3. Clone (copying an entire repo)

Worktrees accomplish a similar goal (allow you to work on different changes without losing work), but are particularly useful when:

1. You want to switch back and forth between the two change sets without having to run a bunch of `git` commands (not branches or stash)
2. You want to keep a light footprint on your machine that's still connected to the main repo (not clone)

### The Main Worktree

- Contains the `.git` directory with the entire state of the repo
- Heavy (lots of data in there!). To get a new main working tree requires a `git clone` or `git init`

### A Linked Worktree

- Contains a `.git` _file_ with a _path_ to the main working tree
- Light (essentially no data in there!), about as light as a branch
- Can be complicated to work with when it comes to env files and secrets

### Create a Linked Worktree

To [create a new worktree](https://git-scm.com/docs/git-worktree#Documentation/git-worktree.txt-addltpathgtltcommit-ishgt) at a given path:

```sh
git worktree add <path> [<branch>]
```

Adding `<branch>` is optional. It will use the last part of the path as the branch name.

Note:
Linked worktrees behave just like a "normal" `git` repo. You can create new branches, switch branches, delete branches, create tags, etc etc.

_BUT_ there is one thing you **cannot** do... you cannot work on a branch that is currently checked out by any other working tree (main or linked).

### Upstream

When you make a change in a _linked_ worktree, that change is automatically reflected in the _main_ worktree!

It makes sense: the linked worktree doesn't have a `.git` directory, so it's not a separate repository. It's just a different view of the _same_ repository.

You can almost think of a linked worktree as just another branch in the same repo, but with its own space on the filesystem.

### Delete Worktrees

Stash is still useful for tiny stuff, but worktrees are so much better for long-lived changes.

However, at some point you will need to clean up your worktrees. The simplest way is the [`remove` subcommand](https://git-scm.com/docs/git-worktree#Documentation/git-worktree.txt-remove):

```sh
git worktree remove WORKTREE_NAME
```

An alternative is to delete the directory manually, then [`prune`](https://git-scm.com/docs/git-worktree#Documentation/git-worktree.txt-prune) all the worktrees (removing the references to deleted directories):

```sh
git worktree prune
```


## Tags

A tag is a name linked to a commit that doesn't move between commits, unlike a branch. Tags can be created and deleted, but not modified.

### How to Tag

To list all current tags:

```sh
git tag
```

To create a tag on the current commit:

```sh
git tag -a "tag name" -m "tag message"
```

### Semver

["Semver"](https://semver.org/), or "Semantic Versioning", is a naming convention for versioning software. You've probably seen it around, it looks like this:

![](https://storage.googleapis.com/qvault-webapp-dynamic-assets/course_assets/l9sosco-1280x591.png)

It has two primary purposes:

1. To give us a standard convention for versioning software
2. To help us understand the impact of a version change and if it's safe (how hard it will be) to upgrade to

Each part is a number that starts at 0 and increments upward forever. The rules are simple:

- MAJOR increments when we make "breaking" changes (this is typically a big release, for example, Python 2 -> Python 3)
- MINOR increments when we add new features in a backward-compatible manner
- PATCH increments when we make backward-compatible bug fixes

To sort them from highest to lowest, you first compare the major versions, then the minor versions, and finally the patch versions. For example, a major version of `2` is always greater than a major version of `1`, regardless of the minor and patch versions.

_As a special case, major version `0` is typically considered to be pre-release software and thus the rules are more relaxed._

### Push tags to Github

```
 git push origin --tags
```