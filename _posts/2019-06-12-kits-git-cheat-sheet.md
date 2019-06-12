---
title: Kit's Git Cheat Sheet [git]
---

## Repo Creation

`cd` to directory

    git init

or

    git init repo_dir

"Bare" repository creation (e.g. for use in network file-based share)

    git init --bare

Clone an existing repository (default name of first remote is `origin`)

    git clone repo_url

## Changes

State of changed files in working directory

    git status

Changes to tracked files

    git diff

Changes to files that are added to index

    git diff --cached

Add all current changes

    git add .

Add some changes in \<file> to the next commit

    git add -p <file>

Commit all local changes in tracked files (quick-add and commit)

    git commit -a

Commit staged

    git commit

Change last commit with staged

    git commit --amend

## History

List commits, newest to oldest

    git log

List commits within a range of revisions (rev identifiers can be branches, revision id's, tags, etc.)

    git log <from>..<to>

Show changes to a file

    git log -p <file>

Show changes, excluding merge commits

    git log --no-merged

Show changes

Kit's "tab-separated changes we're about to release with author name" one-liner

    git log prod..test --pretty=format:"%h%x09%an%x09%s" --no-merges

Annotate file

    git blame <file>

## Branches & Tags

List all branches with last commit message, and tracking branch info

    git branch -avv

Switch `HEAD` to a branch (note that this will check out remote branch and track if ref matches)

    git checkout <branch>

Create new branch based on current `HEAD` and check it out

    git checkout -b <new-branch>

Delete local branch (`-D` to force)

    git branch -d <branch>

Delete branch on remote `origin`

    git push origin --delete <branch>

List branches on remote that are not merged into specified branch on remote

    git branch -r --no-merged origin/<branch>

Create a (lightweight) tag on the current commit

    git tag <tag-name>

Push a specific tag to remote `origin`

    git push origin <tag-name>

Push all tags to current default remote (not usually recommended)

    git push --tags

Get a cool "tag plus number of commits plus short revision id" value for current head

    git describe --tags

## Update & Publish

Get latest branch info from remote and delete tracking branches no longer present on remote

    git fetch --prune

Push local branch to remote and track (`--set-upstream`) (Note, no need to checkout branch you're pushing if specified in the command)

    git push -u origin <branch>

## Merge & Rebase

Merge \<branch> into your current `HEAD`

    git merge <branch>

Rebase your current `HEAD` into \<branch> (think opposite order of operations)

    git rebase <branch>

After rebase is complete, stage changes and commit

In the event of merge conflicts, abort with `git rebase --abort` and `git rebase --continue` after resolving.

## Advanced

Move last commit intended for a **new** branch from current branch to \<new-branch>

    git branch <new-branch>
    git reset HEAD~ --hard
    git checkout <new-branch>

Move last commit to another existing branch

    # undo the last commit, but leave the changes available
    git reset HEAD~ --soft
    git stash
    # move to the correct branch
    git checkout name-of-the-correct-branch
    git stash pop
    git add . # or add individual files
    git commit -m "your message here"
    # now your changes are on the correct branch

or use cherry-pick

    git checkout name-of-the-correct-branch
    # grab the last commit to master
    git cherry-pick master
    # delete it from master
    git checkout master
    git reset HEAD~ --hard
