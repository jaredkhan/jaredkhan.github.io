---
layout: post
title:  "How Git LFS Works"
date:   2020-10-31 18:14:00 +0000
tags: Git
---

*This post covers the mechanisms that [Git LFS (Large File Storage)](https://git-lfs.github.com) uses internally to do its job:*

- *Git subcommands to give us the `git lfs` command in the first place*
- *Git clean and smudge filters to replace large files with pointer files*
- *Git pre-push hooks to push the files to the server*
- *SHA-256 hashes to identify file contents*

*This post is not intended as a general introduction to using Git LFS, but can help us understand how it works.*

## The Problem Git LFS Solves

Imagine you work in a team that uses Git and your Git repository contains, say, a 10MB file that changes often. Perhaps the repo is a web application and the file is a screenshot of the app, that gets updated with every commit. 

Git needs to remember every version of every file in the repository. It starts by storing all the different versions of the file separately in full, it calls these 'loose objects'. When it notices that there are lots of loose objects it will do something called 'packing', where it remembers one revision of the file as well as the differences between that and the other versions. This saves a lot of space if the differences are small relative to the size of the file. See [Git Internals - Packfiles](https://git-scm.com/book/en/v2/Git-Internals-Packfiles) for more.

Let's assume that the file almost entirely changes each time and so even these differences take up a lot of space. With 100 commits of this 10MB file, each team member has to download ~1GB to their local machine even though they rarely work with historical versions of this large file. Ideally, they would only have the versions of this file that they actually need to check out. 

Git LFS does not save space on your server, but saves you space on your local copies of the repo.

## How Git LFS Is Used

The following instructions can be found on the [Git LFS website](https://git-lfs.github.com):

The remote Git server must be set up to support the Git LFS API. 
This is done for you on GitHub, GitLab (requires some configuration) and Bitbucket. 

Each user of the repository must:

- Download the `git-lfs` executable
- Run `git lfs install` on their machine

One user of the repository must:

- Run `git lfs track "*.psd"`, replacing "*.psd" with the filename pattern that you want to track
- Make sure the `.gitattributes` file is checked into the repository

Now all users can add files as normal and Git LFS will work away in the background.

## What Git LFS Does

Git LFS replaces the large files that you try to `git add` for commit in the repo with 'pointer files', files that just contain an identifier of the content, and it stores the content files themselves in a separate local folder, the Git LFS Cache at `.git/lfs/objects`. 

When you `git push` your commits, the new large files in the local Git LFS Cache are uploaded to the server separately. 
This is done using the Git LFS API that the server must implement.  

Whenever you do a `git checkout`, Git LFS will find all the pointer files and replace them with the files themselves, downloading whatever files necessary from the remote server. This means that you only store locally the large files that you actually check out or that you committed yourself, not **all** versions of large files in the history of the repo.

So Git LFS needs to intercept `add`, `push` and `checkout` in order to work and needs to keep track of which pointer files map to which large files in the Git LFS Cache. Understanding how it does these things in more detail can give us more confidence when using Git LFS and guide us in the right direction when something goes wrong.

## How Git LFS Works

### Identifying File Contents Using Hashes

A file name is not a very good way to identify the content of a file. I can have two very different files with the same name or I can change the name of a file without changing its content. Git LFS doesn't use file names to identify files, but uses hashes instead. The *hash* of a file is something that uniquely identifies the contents of the file. An algorithm called SHA-256 exists to create such identifiers.

One is incredibly unlikely to find two *different* files with the same SHA-256 hash and the SHA-256 hash of a particular file doesn't change unless its content does. This makes the SHA-256 hash a much better way to identify the contents of a file than the file name.

For example, the SHA-256 hash of a file with just the text 'hello' in it can be computed like so:

```bash
echo "hello" > hello.txt
sha256sum hello.txt
# 5891b5b522d5df086d0ff0b110fbd9d21bb4fc7163af34d08286a2e846f6be03  
# hello.txt
```

For more discussion on hashing, see [Hashing Algorithms and Security - Computerphile](https://www.youtube.com/watch?v=b4b8ktEV4Bg)

### Clean and Smudge filters

Git has the concept of the *staging area* (or *index*) where changes go before they are committed. We select changes to place into the staging area with `git add`. Git has a feature called *filters* which let us process files just before they are staged (the 'clean' filter) and process files just before they are checked out into your working tree (the 'smudge' filter).

These filters can be used for things like:

- keeping any passwords or other secrets out of the repository
- including the last-modified date of a file in the file itself
- pulling in files from non-Git sources

**How a filter is set up**

Understanding how filters are set up can help us understand what's happening when we install Git LFS.

First, the clean and smudge actions for the filter need to be added to either the user's `~/.gitconfig` file or the repository-local `.git/config` file. **This needs to be done on each user's machine.** Here, we'll add a filter that censors the word 'bottoms', because we can't be having such *foul* language getting checked into our repository:

```bash
[filter "hide-naughty-word"]
  clean = sed s/bottoms/b------/
  smudge = sed s/b------/bottoms/
```

Here, I assume that all users of the repository already have the `sed` command installed on their machine.

Then, we need to tell Git which files to run this filter on by adding to the `.gitattributes` file in the repo. Here, we will run the `hide-naughty-word` filter on all files (`*`):

```bash
*  filter=hide-naughty-word
```

The `.gitattributes` file can be checked into the repo so that it only needs to be written once.

Now we can see this new filter in action by committing a new file and then using `git cat-file` to see what has actually ended up in the repository:

```bash
echo "i like big bottoms and i cannot lie" > naughty.txt
git add naughty.txt
git commit -m "Add naughty file"
git cat-file blob "HEAD:naughty.txt"
# i like big b------ and i cannot lie
cat naughty.txt
# i like big bottoms and i cannot lie
```

We can see that the content has been filtered in the repository but when we actually view the *checked out* file (for example, by using the `cat` tool) it has the content we expect.

**Running filters on files that are already committed**

What if someone on our team doesn't have the filters set up properly and checks in an unfiltered file:

```bash

echo "" > .gitattributes  # empty the .gitattributes file, disabling filter
echo "i like big bottoms and i cannot lie" > naughty2.txt
git add naughty2.txt
git commit -m "Add unfiltered naughty file"
echo "*  filter=hide-naughty-words" > .gitattributes  # reinstate filter
git cat-file blob "HEAD:naughty2.txt"
# i like big bottoms and i cannot lie
```

We can re-run the filters on all files and create a new commit like so:

```bash
git add --renormalize .
git commit -m "Run filters against all files"
git cat-file blob "HEAD:naughty2.txt"
# i like big b------ and i cannot lie
```

Note that the unfiltered file still exists in the history.

### Git subcommands

Any executable available in your path that starts with `git-` can be used as a git 'subcommand'.

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

### **Pre-push hooks**

Git has all sorts of points you can hook into to trigger actions based on Git events. These are a set of scripts in the `.git/hooks` directory which are run when specific Git events happen. These scripts cannot be checked into the repo so need to be put in place on each individual user's machine.

One hook of particular interest for Git LFS is the pre-push hook. This script lives at the path `.git/hooks/pre-push` and takes as input the branch names being pushed from/to as well as a list of commits being pushed.

The script can do whatever it likes with this list of commits and then exit. If it exits with a non-zero exit code, the push will be cancelled.

### Bringing it all together

When you install Git LFS, you will get an executable called `git-lfs`. Because it's named starting with `git-`, it is also now runnable with `git lfs`. When you run `git lfs install` on your machine, your `~/.gitconfig` file is updated to contain the Git LFS filter definition.

```bash
[filter "lfs"]
	clean = git-lfs clean %f
	smudge = git-lfs smudge %f
	required = true
	process = git-lfs filter-process
```

This needs to be run once by each user of the repository before they can successfully use Git LFS.

When you run, for example, `git lfs track "*.jpg"` to track all .jpg files in the repo with Git LFS, your `.gitattributes` file is updated:

```bash
*.jpg filter=lfs diff=lfs merge=lfs -text
```

 This tells Git to use the `lfs` clean and smudge filter for these files, as well as attaching some extra attributes. With the filters in place, whenever you stage a `.jpg` file it will be replaced with a pointer file containing the SHA-256 hash of the file content. The file itself gets stored in `.git/lfs/objects` at a path based on the hash so that it's easy to find later.

Almost every Git LFS command you run (including the clean and smudge filters) will also modify the pre-push hook if it's not already set up. So, because we ran some Git LFS commands already, the `.git/hooks/pre-push` file should already look like this:

```bash
#!/bin/sh
command -v git-lfs >/dev/null 2>&1 || { echo >&2 "\nThis repository is configured for Git LFS but 'git-lfs' was not found on your path. If you no longer wish to use Git LFS, remove this hook by deleting .git/hooks/pre-push.\n"; exit 2; }
git lfs pre-push "$@"
```

We can see it just forwards to the `git lfs pre-push` command. Git LFS's pre-push command will read the list of commits to be pushed and scan each commit for new pointer files. For each new pointer file it finds, it looks up the actual large file in the Git LFS Cache. It will then upload all the new files to the remote server using the Git LFS API.

Note that because it's using hashes for file identity, you will never end up with two copies of the same large file either locally or on the remote server.

And that's it! These are the main mechanisms Git LFS uses to do its job.

## What could possibly go wrong?

Now that we understand the main mechanisms in use, we can debug some issues that might arise:

- "I'm seeing pointer files when I should be seeing the actual files"
    - The smudge filter is not properly set up for this file.
        - Ensure the .gitattributes file has a line for this file with `filter=lfs`
        - Ensure you have the filter installed. It's perfectly harmless to re-run `git lfs install`
        - Check out the file again to re-run the filter: `git checkout -- <path-to-file>`
- "Large files are being checked in when I wanted pointers to be checked in"
    - The clean filter is not properly set up for this file.
        - Follow a similar remedy to above
        - Re-run the filter with `git add --renormalize <path-to-file>`

## Summary

Git LFS:

- Does not save you any space on the remote server, but saves you space locally
- Uses clean and smudge filters
- Installs a `git-lfs` executable which, due to its name, can also be run with `git lfs`
- Installs filters in `~/.gitconfig` when you run `git lfs install`
- Configures filters in `.gitattributes` when you run `git lfs track ...`
- Configures the pre-push hook whenever you run any `git-lfs` command (including the clean and smudge filters)
- Puts large files away in `.git/lfs/object`, named by their SHA-256 hashes, using the clean filter
- Pushes large files to the remote using the pre-push hook

## References

- [Git LFS Client Specification](https://github.com/git-lfs/git-lfs/blob/master/docs/spec.md)
- [Tim Pettersen — Tracking huge files with Git LFS](https://www.youtube.com/watch?v=w-037RcHjAA)
- [BitBucket Git LFS Tutorials](https://www.atlassian.com/git/tutorials/git-lfs) — These are well-written and cover many practical details of using Git LFS
- [Git Hooks Documentation](https://git-scm.com/docs/githooks)
- [Git Attributes Documentation](https://git-scm.com/docs/gitattributes)
- [Git LFS Source Code](https://github.com/git-lfs/git-lfs)