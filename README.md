# How to remove sensitive data that has been committed and pushed in Git

Even though we try to keep our `.env` files safe and in no way commit sensitive data to our repos, sometimes it happens and we are faced with the fact afterwards, trying to fix the damage.

Important: the safest way to resolve situation above is to immediately rotate the secret that got exposed, but this repo is created as a speculation on how we can fix this with Git only.

## How to rewrite history
There are several ways to rewrite history:
1. Amending last commit
```
git commit --amend
```
2. Remove the commit, but keep all the changes in the working copy.
```
git reset --soft HEAD~1
```
3. Remove the last commit completely with `reset --hard`.  
**Please, make sure you know what you're doing since this will remove files in your local copy and all the commits specified; and it will be quite hard to get them back.**
```
git reset --hard HEAD~1
```
4. [Rebase interactive](https://git-scm.com/book/en/v2/Git-Tools-Rewriting-History), which lets you change multiple commits in a lot of different ways.
```
git rebase --interactive HEAD~5
```

All of these methods rewrite history and create new commits. If now I would do `git push --force`, my remote at GitHub will be rewritten. Done deal? Well, not exactly.

## How to rewrite history???

When I have amended my sensitive commit and did force push to GitHub, the new commit has been created and it seems like Git history got overwritten in both my local repo and on GitHub. However, if I have saved the hash of my previous sensitive commit, I can still access it on GitHub. Why?

The commits are still there, they are just in the dangling state (not attached to any branch or remote):
```
➜  git-sensitive-data-removal git:(master) git fsck --unreachable --no-reflogs
Checking object directories: 100% (256/256), done.
Checking objects: 100% (3/3), done.
unreachable blob c7104b575ecae75e7ef360c27f36c9c77aa0c43f
unreachable tree 283ff78e0bef05729dca3df490d197d9b0827e59
unreachable commit b006ba9abacaaae351bbcefd4bcda9e2f8043c5b
unreachable commit 5a00f49f66b767989f5f76af8e58a090558fe6a0
```

Git does this to let you change your mind and go back to your "deleted" commits. After some time, the commit will be picked up by Git garbage collector and removed. However, in this case this is still unsafe for us. If someone somehow cached the commit hash, they will still be able to see my sensitive password, even though technically it's not there.

To be able to thoroughly clean my commit history and make sure that sensitive data is removed, I can run gc manually (here's StackOverflow [answer](https://stackoverflow.com/questions/3765234/listing-and-deleting-git-commits-that-are-under-no-branch-dangling)):

```
git reflog expire --expire-unreachable=now --all
git fsck --unreachable
git gc --prune=now
```

!!! While this would clean all my dangling commits, it can do a lot of damage, since `git reflog expire --expire-unreachable=now --all` makes all my stashes expired and as soon as `git gc` has run, I will no longer be able to see any of them.

So what you would want to do first is to make sure you don't have important changes stashed away, and all the stashed changes are committed to a branch.

However, what I have found is that even with removing those dangling commits, I can still access them on GitHub.
This if what GitHub team [writes](https://help.github.com/en/articles/removing-sensitive-data-from-a-repository) in the note on removing commits:

> Warning: Once you have pushed a commit to GitHub, you should consider any data it contains to be compromised. If you committed a password, change it! If you committed a key, generate a new one.
> 
> This article tells you how to make commits with sensitive data unreachable from any branches or tags in your GitHub repository. However, it's important to note that those commits may still be accessible in any clones or forks of your repository, directly via their SHA-1 hashes in cached views on GitHub, and through any pull requests that reference them. You can't do anything about existing clones or forks of your repository, but you can permanently remove cached views and references to the sensitive data in pull requests on GitHub by contacting GitHub Support or GitHub Premium Support.

## Removing sensitive file once and for all with BFG?

Another alternative GitHub itself lists as a possible solution is using [BFG tool](https://rtyley.github.io/bfg-repo-cleaner/) for removing sensitive files from repo, which apparently let's you unreference sensitive files easier.

At the same time, it seems like even re-creating your repo from scratch doesn't guarantee you that the secret is safe and sound, so I would still recommend to rotate it.

## Summary

In the end, the easiest thing to avoid all this is really make sure that you `.gitignore` sensitive files by default or get your credentials from ENV, so that they don't end up in your remotes under any circumstances.

By the way, did you know about `git add -p`? It lets you stage changes in your tracked files chunk by chunk, while you can review everything you're adding to staged changes.
