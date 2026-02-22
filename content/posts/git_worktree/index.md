---
title: "Saving myself n^2 fetches and n^3 headaches with git-worktree"
date: 2026-02-21T16:36:52-08:00
draft: false
toc: false
images:
tags:
  - git
---

I recently discovered the `git worktree` feature, a handy tool for setting up multiple working trees for your git repository.
While in retrospec, it might seem obvious, it falls into that category of `git` knowledge that you only have after gaining some real experience with the tool.
I will introduce how I use this feature by demonstrating how it solved two problems for me.

# Problem 1: Too Many Remotes
As part of my job, I part of a team tasked with maintaining a fork of the Linux kernel.
Actually, forks plural, since every few minor Linux versions, we rebase our ~1500 patch series to create a new release, and hence a new fork.
This could be done with branches off of one tree, but (a) Linux kernel trees typically operate with "branches as features", and "trees as release candidates", and (b) this way, for each release, we are always deploying from `master`.

This rebase process, as well as the general task of parallel development across these kernels, was fairly painful.
Outside of the issue of porting all 1500 patches to create a release, the actual tooling can be a bit messy, if you are a bit naive.
For quite a (embarassingly) long time, I was creating a separate local repository for each of these releases.
So, for something as simple as porting one single fix from one release to another, the process looked something like:

1. Use `git format-patch` to create a patch containing the fix.

2. Apply the patch to the other tree with `git am`.
Since it is highly likely that the fix depends on other out-of-tree commits, a 3-way apply will probably be needed.

This isn't *terrible*, and certainly was good enough for us to use productively for a while.
However, it is tedious, especially when attempting to port over hundreds of commits at a time, and the 3-way apply can be dubious if the context is just crappy enough.

The next, obvious solution, is to (mostly) get rid of the need for the 3-way apply by fetching the first repository from the second.
In fact, this gets rid of the need to use patchfiles altogether, as `git cherry-pick` can be used, since you will have a reference to the raw commits.

*However*, this solution is where the n\^2 in the title comes into play.
If I have n repositories, and need to potentially cherry-pick commits between them, then I will need to fetch n\^2 times.
This is only, like, n=4 at a maximum for actively maintained repositories for me.
Still, though that would be 16 fetches, most of which are duplicate work, not to mention fetches of other Linux trees that I might care about that I do not maintain.

Enter `git worktree`, my savior.
The basic premise is that the actual files that you see in a git repository outside of the `.git` directory, a.k.a. the *working tree*, really is just rendered from objects contained within that `.git` directory.
Therefore, you can construct multiple working trees from the same local repository, just by asking the repository to render a tree from a particular hash.
This is basically what `git worktree` does.

From each of my working trees, the experience is the same as when I was manually fetching every repo from every repo.
However, instead of constantly updating all of these fetches for each local repo, I only need to fetch once from each remote, for the one single local repo.
The main work is just in the setup, when a few things need to be done.

1. Add remotes for all of my release repositories.

2. Add a local branch to track each of these remotes as upstream.
This is because, while the remotes all have a "master" branch, I need to be able to distinguish them locally.

3. Create a local working tree for each of these remote repos, and checkout the corresponding local branch for each of them.

Step 2 and 3 actually can be combined into the same worktree command!
```bash
git worktree add --track -b <remote>_b path/to/local/worktree <remote>/master
```

As a convention, I find it easiest to name my local branch after the remote that it is tracking, plus some suffix.

And like that, it is done!
However, having multiple remotes is not the only problem that `git worktree` solved for me.

# Problem 2: Too Many Build Targets

Another issue that I run into at work is building for different targets.
We support many different versions of several operating systems, and building for each of them requires (for a few reasons) a sterile working tree for that OS.
I run all of my builds for the Linux-based OS's using Docker on my local machine.
My previous workflow for this just involved separate local repositories for each of these builds, a silly idea now that I am aware of `git worktree`.

Now, I keep one local repository, and have several working trees off of it, one for each build.
There is a bit of a snafu here - `git worktree` disallows multiple working trees to point to the same branch, likely to avoid dealing with how to update each working tree when one changes the branch `HEAD`.
However, creating a branch for each working tree and manually rebasing each onto my `master` branch when needed is hardly tedious, and this actually lets me test my changes across multiple OS's before pushing without creating more ugly patchfiles!

```bash
git worktree add --track -b <os_name>_b <os_name>_wt origin/master
```
