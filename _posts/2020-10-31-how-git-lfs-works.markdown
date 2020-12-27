---
layout: post
title:  "How Git LFS Works"
date:   2020-10-31 18:14:00 +0000
tags: Git
---

*[Git LFS (Large File Storage)](https://git-lfs.github.com) helps you version large files in Git without having to download every version of it. This post explains the mechanisms that Git LFS uses internally to do its job, including:*

- *Git subcommands to give us the `git lfs` command in the first place* [[Jump]](#git-subcommands)
- *Git clean and smudge filters to replace large files with pointer files* [[Jump]](#clean-and-smudge-filters)
- *Git pre-push hooks to upload the large files to a server* [[Jump]](#pre-push-hooks)

## The Problem Git LFS Solves

Git needs to remember every version of every file in the repository. It starts by storing **all the different versions of the file separately in full**, it calls these 'loose objects'. 

When it notices that there are lots of loose objects it will do something called 'packing', where it remembers one version of the file as well as the differences between that and the other versions. This saves a lot of space if the differences are small relative to the size of the file. See [Git Internals - Packfiles](https://git-scm.com/book/en/v2/Git-Internals-Packfiles) for more.

Even with packing, if a repo has a large file that changes often then this can quickly require a lot of storage space to make every version available locally, even though you might rarely work with historical versions of this large file. Ideally, you would only download and store the versions of this file that you actually need to view or work with, but you don't want that to get in the way of your normal Git workflow. 

Git LFS does not save space on your server, but saves you space on your local copies of the repo.

## How Git LFS Is Used

The following instructions can be found on the [Git LFS website](https://git-lfs.github.com):

The remote Git server must be set up to support the [Git LFS API](https://github.com/git-lfs/git-lfs/blob/7b8779f3d46350d450394222d81e140a5a98911e/docs/api/batch.md#L1). 
This is done for you on GitHub, GitLab (requires some configuration) and Bitbucket. 

Each user of the repository must:

- Download the `git-lfs` executable
- Run `git lfs install` on their machine

One user of the repository must:

- Run `git lfs track "*.psd"`, replacing "*.psd" with the filename pattern that you want to track
- Make sure the `.gitattributes` file is checked into the repository

Now all users can add files to the repo as normal and Git LFS will work away in the background.

## What Git LFS Does

There are three main things that Git LFS does for us:

1. Git LFS replaces the large files that you try to `git add` for commit in the repo with *pointer files*, files that just contain an identifier of the content, and it stores the content files themselves in a separate local folder, the Git LFS Cache at `.git/lfs/objects`. 

2. When you `git push` your commits, the new large files in the local Git LFS Cache are uploaded to the server separately. 
This is done using the Git LFS API that the server must implement.  

3. Whenever you do a `git checkout`, Git LFS will find all the pointer files and replace them with the files themselves, downloading whatever files necessary from the remote server. This means that you only store locally the large files that you actually checkout or that you committed yourself, not **all** versions of large files in the history of the repo.

To perform these three magic tricks, Git LFS needs to intercept `add`, `push` and `checkout` and needs to keep track of which pointer files map to which large files in the Git LFS Cache. Understanding how it does these things in more detail can give us more confidence when using Git LFS and guide us in the right direction when something goes wrong.

## How Git LFS Works

### Git subcommands

`git clone` and `git push` are two different built-in *subcommands* of `git`. As it turns out, any executable available in your `PATH` that starts with `git-` can be used as a Git subcommand.

For example:

```bash
# creating an executable with the name 'git-shout'
echo '#! /usr/local/bin/bash\necho "running custom command!"' > git-shout
chmod u+x git-shout
# and making sure it's in the shell's PATH
export PATH=$PATH:.

git-shout
# running custom command!

# but it can also be called like this:
git shout
# running custom command!
```

There's nothing special about it being a subcommand, it's just a nice-looking alias to the same executable.


### Clean and Smudge filters

Git has the concept of the *staging area* (or *index*) where changes go before they are committed. We select changes to place into the staging area with `git add`. Git has a feature called *filters* which let us process files just before they are staged (the 'clean' filter) and process files just before they are checked out into your working tree (the 'smudge' filter).

These filters can be used for things like:

- keeping any passwords or other secrets out of the repository
- including the last-modified date of a file in the file itself
- pulling in files from non-Git sources

#### Creating a new filter

First, the clean and smudge actions for the filter need to be added to either the user's `~/.gitconfig` file or the repository-local `.git/config` file. Either way, **this needs to be done on each user's machine.** As a simple example, we'll add a filter that censors the word 'butts', because we can't be having such *foul* language getting checked into our repository:

```bash
# .git/config

# define a filter called 'hide-naughty-word'
[filter "hide-naughty-word"]
  # define the command for this filter 
  clean = sed s/butts/b--ts/
  smudge = sed s/b--ts/butts/
```

Here, I assume that all users of the repository already have the `sed` command installed on their machine.

#### Assigning the filter to files/filetypes

Then, we need to tell Git which files to run this filter on by adding to the `.gitattributes` file in the repo. We can assign the filter to a specific file (`my-specific-file.txt`), a specific file type (`*.txt`), or all files (`*`). For more detail see [gitattributes Documentation](https://git-scm.com/docs/gitattributes). Here, we will run the `hide-naughty-word` filter on all files:

```bash
*  filter=hide-naughty-word
```

This `.gitattributes` file can be checked in so that it can be shared amongst all users of repo.

Now we can see this new filter in action by committing a new file and then using a command called `git cat-file` to see what has actually ended up in the repository:

```bash
echo "i like big butts and i cannot lie" > mix-a-lot.txt
git add mix-a-lot.txt
git commit -m "Add naughty file"
git cat-file blob "HEAD:mix-a-lot.txt"
# i like big b--ts and i cannot lie
cat mix-a-lot.txt
# i like big butts and i cannot lie
```

We see that the content has been filtered in the repository but when we actually view the *checked out* file (for example, by using the `cat` tool) it has the content we expect.

#### Running filters on files that are already committed

What if someone on our team doesn't have the filters set up properly and checks in an unfiltered file:

```bash

echo "" > .gitattributes  # empty the .gitattributes file, disabling filter
echo "i like big butts and i cannot lie" > mix-a-lot2.txt
git add mix-a-lot2.txt
git commit -m "Add unfiltered naughty file"
echo "*  filter=hide-naughty-words" > .gitattributes  # reinstate filter
git cat-file blob "HEAD:mix-a-lot2.txt"
# i like big butts and i cannot lie
```

We can re-run the filters on all files and create a new commit like so:

```bash
git add --renormalize .
git commit -m "Run filters against all files"
git cat-file blob "HEAD:mix-a-lot2.txt"
# i like big b--ts and i cannot lie
```

Note that the unfiltered file still exists in the history.

### Pre-push hooks

'Git hooks' are a way to run custom scripts when certain events happen. These scripts can be added to the `.git/hooks` directory of your repo with names like `pre-commit` or `post-checkout` and can be modified to do whatever you like.
The list of hooks that you can set up can be found in [git/Documentation/githooks.txt](https://github.com/git/git/blob/e31aba42fb12bdeb0f850829e008e1e3f43af500/Documentation/githooks.txt).

For example, when you run `git push`, Git will first run the `pre-push` script (if it exists). If that script exits with a non-zero exit code, the push will be aborted.

Git hooks cannot be checked into a repo. If all users of a project need to run the same Git hooks, each individual user will need to set them up on their copy of the repo.


### Bringing it all together

When you install Git LFS, you will get an executable called `git-lfs`. Because it's named starting with `git-`, it is also now runnable with `git lfs`. When you run `git lfs install` on your machine, your `~/.gitconfig` file is updated to contain the Git LFS filter definition.

```bash
[filter "lfs"]
  clean = git-lfs clean %f
  smudge = git-lfs smudge %f
  required = true
  process = git-lfs filter-process
```

Because this is modifying a user-specific file, `git lfs install` needs to be run once by each user of the repository before they can successfully use Git LFS.

When you run, for example, `git lfs track "*.jpg"` to track all .jpg files in the repo with Git LFS, it updates your `.gitattributes` file:

```bash
*.jpg filter=lfs diff=lfs merge=lfs -text
```

 This tells Git to use the `lfs` clean and smudge filter for these files, as well as attaching some extra attributes. With the filters in place, whenever you stage a `.jpg` file it will be replaced with a pointer file containing the SHA-256 hash of the file content. The file itself gets stored in `.git/lfs/objects` at a path based on the hash so that it's easy to find later.
Note that the .gitattributes file can and should be checked into the repository so that everyone on the project tracks the same files in Git LFS.

Almost every Git LFS command you run (including `git lfs install` and the clean and smudge filters) will also modify the pre-push hook if it's not already set up. So, because we ran some Git LFS commands already, the `.git/hooks/pre-push` file should already look like this:

```bash
#!/bin/sh
command -v git-lfs >/dev/null 2>&1 || { echo >&2 "\nThis repository is configured for Git LFS but 'git-lfs' was not found on your path. If you no longer wish to use Git LFS, remove this hook by deleting .git/hooks/pre-push.\n"; exit 2; }
git lfs pre-push "$@"
```

It first checks if the `git-lfs` executable exists and gives an error message if not. It then forwards to the `git lfs pre-push` command. Git LFS's pre-push command will read the list of branches to be pushed and scan each new commit in those branches for new pointer files [[source code]](https://github.com/git-lfs/git-lfs/blob/8d234c690b94e8a5b12ff72e3fba8481f032daf0/commands/command_pre_push.go#L40). For each new pointer file it finds, it looks up the actual large file in the Git LFS Cache. It will then upload all the new files to the remote server using the Git LFS API.

Note that because it's using hashes for file identity, you will never end up with two copies of the same large file on the remote server.

And that's it! These are the main mechanisms Git LFS uses to do its job.

## What could possibly go wrong?

Now that we understand the main mechanisms in use, we can debug some issues that might arise:

> I'm seeing pointer files when I should be seeing the actual files

The smudge filter might not be properly set up for this file.

- Ensure the .gitattributes file has a line that matches this file with `filter=lfs`
- Ensure you have the filter installed. It's perfectly harmless to re-run `git lfs install`
- Check out the file again to re-run the filter: `git checkout -- <path-to-file>`
  
> Large files are being checked in when I wanted pointers to be checked in

The clean filter might not be properly set up for this file.

- Follow a similar remedy to above
- Re-run the filter with `git add --renormalize <path-to-file>`

> My repo is still taking up loads of space

- Installing Git LFS won't automatically run the LFS filters on historical commits. If you already have large files in your history and want to rewrite your Git history to avoid that, take a look at [`git lfs migrate`](https://github.com/git-lfs/git-lfs/blob/7b8779f3d46350d450394222d81e140a5a98911e/docs/man/git-lfs-migrate.1.ronn)
- If Alice makes 10 commits that change a large file, pushes them, and then Bob checks out her latest commit, then Bob will only download the latest version but Alice will likely still have all 10 versions in her LFS cache. Alice may want to run [`git lfs prune`](https://github.com/git-lfs/git-lfs/blob/7b8779f3d46350d450394222d81e140a5a98911e/docs/man/git-lfs-prune.1.ronn) in her copy of the repo to get rid of unneeded versions of the file.

## Summary

In summary, Git LFS...

- ...does not save you any space on the remote server, but saves you space locally.
- ...uses clean and smudge filters.
- ...installs a `git-lfs` executable which, due to its name, can also be run with `git lfs`.
- ...installs filters in `~/.gitconfig` when you run `git lfs install`.
- ...configures filters in `.gitattributes` when you run `git lfs track ...`.
- ...configures the pre-push hook whenever you run any `git-lfs` command (including the clean and smudge filters).
- ...puts large files away in `.git/lfs/object`, named by their SHA-256 hashes, using the clean filter.
- ...pushes large files to the remote using the pre-push hook.

## References

- [Git LFS Client Specification](https://github.com/git-lfs/git-lfs/blob/master/docs/spec.md)
- [Tim Pettersen — Tracking huge files with Git LFS](https://www.youtube.com/watch?v=w-037RcHjAA)
- [BitBucket Git LFS Tutorials](https://www.atlassian.com/git/tutorials/git-lfs) — These are well-written and cover many practical details of using Git LFS
- [Git Hooks Documentation](https://git-scm.com/docs/githooks)
- [Git Attributes Documentation](https://git-scm.com/docs/gitattributes)
- [Git LFS Source Code](https://github.com/git-lfs/git-lfs)