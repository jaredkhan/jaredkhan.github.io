---
layout: post
title:  "Running Mypy in Pre-commit"
date:   2020-12-14 22:05:00 +0000
tags: Git Python
---

*The only thing worse than not type-checking your code is thinking you are type-checking it when you aren't.*

This post is about running Mypy in a Git pre-commit hook using the [Pre-commit](https://pre-commit.com/) framework.  Running Mypy is a little fiddly in itself, and [pre-commit/mirrors-mypy](https://github.com/pre-commit/mirrors-mypy) (the de facto way to call Mypy in Pre-commit) calls Mypy in a slightly opinionated way that may introduce more confusions or hide errors you want to see.


**Three take-away points if you're in a hurry:**
- Make sure you run Mypy on all files, not just those that have changed
- Make sure Mypy has access to the installed dependencies of the code it is type-checking
- Be careful with the use of flags that reduce the strictness of Mypy like `--ignore-missing-imports`


Here, I show you how to make your own Mypy hook that suits your needs, in *3 only-somewhat-fiddly steps*:
1. Running Mypy correctly outside of Pre-commit [[Jump]](#step-1-running-mypy-correctly-outside-of-pre-commit)
2. Creating your own Pre-commit hook [[Jump]](#step-2-creating-our-own-pre-commit-hook)
3. Giving Mypy access to your project dependencies [[Jump]](#step-3-giving-the-mypy-hook-access-to-dependencies)

## A solution that works in my case

Before discussing the gory details and alternatives, here's a solution that works for my project.

I add a `mypy.ini`:

```toml
[mypy]
# mypy_path will vary (and may not be necessary) 
# for your project layout.
mypy_path=./src:./tests

# Explicitly blacklist modules in use
# that don't have type stubs.
[mypy-pytest.*]
ignore_missing_imports = True
[mypy-pyproj.*]
ignore_missing_imports = True
```

and then add a script at `./run-mypy`:

```bash
#!/usr/bin/env bash

# A script for running mypy, 
# with all its dependencies installed.

set -o errexit

# Change directory to the project root directory.
cd "$(dirname "$0")"

# Install the dependencies into the mypy env.
# Note that this can take seconds to run.
# In my case, I need to use a custom index URL.
# Avoid pip spending time quietly retrying since 
# likely cause of failure is lack of VPN connection.
pip install --editable . \
  --index-url https://custom-index-url.com/simple \
  --retries 1 \
  --no-input \
  --quiet

# Run on all files, 
# ignoring the paths passed to this script,
# so as not to miss type errors.
# My repo makes use of namespace packages.
# Use the namespace-packages flag 
# and specify the package to run on explicitly.
# Note that we do not use --ignore-missing-imports, 
# as this can give us false confidence in our results.
mypy --package acme --namespace-packages
```

and then define a custom Pre-commit hook that runs that script in: `./.pre-commit-config.yaml`

```yaml
# .pre-commit-config.yaml

repos
- repo: local
  # We do not use pre-commit/mirrors-mypy, 
  # as it comes with opinionated defaults 
  # (like --ignore-missing-imports)
  # and is difficult to configure to run 
  # with the dependencies correctly installed.
  hooks:
    - id: mypy
      name: mypy
      entry: "./run-mypy"
      language: python
      # use your preferred Python version
      language_version: python3.7
      additional_dependencies: ["mypy==0.790"]
      types: [python]
      # use require_serial so that script
      # is only called once per commit
      require_serial: true
      # Print the number of files as a sanity-check 
      verbose: true
```

You'll have to adapt this to your own project structure and strictness/performance needs.
To expose all the issues this tries to cover, we'll build it up in 3 steps.

## Step 1: Running Mypy correctly outside of Pre-commit

Before thinking about Pre-commit, we should make sure we can run Mypy directly in the desired way.

### Running on the correct files

Running `mypy .` in the root of your project will often **not** do what you need it to. You should play around, keeping an eye on Mypy output, to make sure Mypy is running on all the files that you want. This involves choosing:

- Whether to specify the files to type-check as a package, a module, a directory, or a file path
- Whether to specify a MYPYPATH
- Whether to add the `--namespace-packages` option
- What working directory to invoke Mypy from

[Running mypy and managing imports](https://mypy.readthedocs.io/en/stable/running_mypy.html#running-mypy-and-managing-imports) is a helpful section of the documentation for getting this right. Pay extra attention when you are using namespace packages, packages without `__init__.py` files. 

I'm writing this whilst v0.790 is the latest release. Simplifying the calling of Mypy, and its import handling is a current priority for the maintainers. See for example the umbrella issue, [#8584 — Redesign import handling](https://github.com/python/mypy/issues/8584). Various improvements have already been merged to the master branch.

### Following the right rules

Once Mypy is running on the correct files,  you'll want to get it running the right checks for your codebase so that it passes whilst also checking what you want it to check. This may involve:

- Making changes to your codebase to meet new rules that you want to enforce
- Setting various strictness settings. For example: `--no-implicit-optional`, `--disallow-untyped-defs`, `--no-strict-optional` or the umbrella option `--strict`
- Deciding which imported modules to treat as `Any`. Sometimes Mypy will complain that it can't find a certain module or its stubs. This can be indicative that Mypy does not have access to these dependencies, which you should fix (see below), but can also mean the library doesn't have any type stubs. For the latter case, it's sensible to treat those modules as `Any` in a `mypy.ini` file:

  ```toml
  # mypy.ini
    
  [mypy]
  # this section is required
  # you can add a mypy_path here, if you need one.
  
  # example of explicitly ignoring missing stubs
  # for a dependency and its subpackages.
  # This is safer than ignoring everything 
  # with the --ignore-missing-imports option.

  [mypy-pyproj.*]
  ignore_missing_imports = True
  ```

You may even want to temporarily introduce errors in certain files to make sure Mypy will notice them. See also [Mypy docs — No errors reported for obviously wrong code](https://mypy.readthedocs.io/en/stable/common_issues.html#no-errors-reported-for-obviously-wrong-code).

### Bake it into a script

Now that you know precisely how you want to call Mypy, create a script called `run-mypy` that captures the arguments you want to use. For example, in my case, I have a namespace package in the `src/acme` directory, and my script ended up looking like this:

```bash
#!/usr/bin/env bash

set -o errexit

# Change directory to the project root directory.
cd "$(dirname "$0")"

# Because I'm using namespace packages,
# I have used --package acme rather than using 
# the path 'src/acme', which would correctly
# collect my files but erroneously add 
# 'src/acme' to the Mypy search path.
# We only want 'src' in the path so that Mypy
# knows our modules by their fully qualified names.
mypy --package acme --namespace-packages
```

I also had to add a mypy_path in mypy.ini:

```toml
[mypy]
mypy_path=./src
```

## Step 2: Creating our own Pre-commit hook

Now that we know how to run Mypy for our project, we can think about running it in Pre-commit. First, a brief primer on how Pre-commit works so that we can consider what might go wrong.

### How Pre-commit runs hooks

Pre-commit installs each Python hook in a separate virtualenv. Before each commit, the list of staged files is passed to that hook. Any unstaged changes are stashed and only restored after all hooks have run.

#### Problem: Only running on changed files

With Mypy, we probably don't want to pass it just the list of changed files:

- It will miss type errors resulting from but not occurring in the staged changes. For example: if you have changed the definition of a function but not a usage of that function in another file then the usage is now invalid, but won't be checked.
- As mentioned above, you may need more control over how Mypy is invoked anyway.
- Mypy uses an [Incremental Mode](https://mypy.readthedocs.io/en/stable/command_line.html#incremental-mode) by default. It stores calculated type information so re-running on all files after only a few changes doesn't take as long. For faster incremental runs, consider using a long-running [Mypy daemon](https://mypy.readthedocs.io/en/stable/mypy_daemon.html#mypy-daemon).

We'll solve this by using our own `run-mypy` script and ignoring the file list that Pre-commit passes to it.

#### Problem: Running in an isolated virtualenv

Mypy running in a separate virtualenv is also problematic, since it won't have access to all the dependencies installed in your main development environment. This means it can't type check usages of those dependencies. We'll solve this in Step 3.

### Setting up the hook

We can solve both these problems with a properly-configured hook, which we'll set up ourselves. To get started, create a new [Repository-local hook](https://pre-commit.com/#repository-local-hooks) by adding the following to your .pre-commit-config.yaml like so

```yaml
# .pre-commit-config.yaml

repos
- repo: local
  hooks:
    - id: mypy
      name: mypy
      entry: "./run-mypy"
      language: python
      # use your preferred Python version
      language_version: python3.7
      additional_dependencies: ["mypy==0.790"]
      # trigger for commits changing Python files
      types: [python]
      # use require_serial so that script
      # is only called once per commit
      require_serial: true
      # print the number of files as a sanity-check
      verbose: true
```

## Step 3: Giving the Mypy hook access to dependencies

Mypy needs an environment where the dependencies are imported so that it can check for type-errors in their usage. Here's a few options for doing that, with differing levels of convenience and speed:

### Option 1: Use `language: system` to run Mypy in an existing environment 

Replace `language: python` in your hook definition with `language: system`. Remove the `additional_dependencies` line and install Mypy into your environment directly. Now, Pre-commit will not create a separate virtualenv for the hook and will run it in whatever environment you happen to be in when you run `git commit` or `pre-commit run`.  This means you always run Mypy directly in your dev environment, but breaks if any of the developers on the project want to trigger Pre-commit from outside the dev environment. For example, this won't work if using a GUI Git client, as the correct virtualenv probably won't be activated.

### Option 2: Point Mypy to a specific environment with `--python-executable`

If it's possible to automatically figure out the path to the appropriate Python interpreter (the one associated with the existing installation of your dependencies, which may or may not be in a virtual environment), then you can point Mypy to that path using the `--python-executable` option on `mypy`.

### Option 3: Install specific dependencies with the `additional_dependencies` hook option

If you only care about type-checking the usages of a few third-party modules, then you can install those specific modules into the hook environment like so:

```yaml
# .pre-commit-config.yaml

repos
- repo: local
  hooks:
    - id: mypy
    name: mypy
    entry: "./run-mypy"
    language: python
    # Replace with appropriate version
    language_version: python3.7
    # install Mypy, and the dependencies
    additional_dependencies: 
      - "mypy==0.790"
      - "sructlog==20.1.0"
    types: [python]
    # use require_serial so that script
    # is only called once per commit
    require_serial: true
    # Print the number of files as sanity-check 
    verbose: true
```

This is relatively fast as Pre-commit remembers the dependencies it installed in the environment [[source code]](https://github.com/pre-commit/pre-commit/blob/cf604f6b93b8fb7a61ab9ca45e5dedbdb4fd5796/pre_commit/repository.py#L54). The downside is this means duplicating your list of dependencies (at least those that have type stubs).

The additional_dependencies are just sent directly to Pip [[source code]](https://github.com/pre-commit/pre-commit/blob/cf604f6b93b8fb7a61ab9ca45e5dedbdb4fd5796/pre_commit/languages/python.py#L199) so you can happily add, for example, a `--index-url` argument in this array. Just be aware that Pre-commit will only re-run Pip when the list of `additional_dependencies` changes, so don't expect to put "requirements.txt" in this array and have it figure out when you've changed that file.

### Option 4: Running a full `pip install` in the hook

**This is not fast.** The speed we mostly care about is that of running the hook on each commit, not its initial setup. However, running `pip install` takes many seconds even when the dependencies are already installed. It is, however, a pretty reliable and easy way to make sure your dependencies are installed if the performance hit is acceptable to you and your team.

To do this, simply add the appropriate `pip` command into your `run-mypy` script. For example:

```bash
#!/usr/bin/env bash

set -o errexit

# Change directory to the project root directory.
cd "$(dirname "$0")"

# Install the dependencies into the mypy environment.
# Note that this can take seconds to run.
pip install --editable . --no-input --quiet

mypy --package acme --namespace-packages
```

This is the option I have gone for so far since it minimises the likelihood of us making, mistakes without imposing many restrictions on local project setup. For example, teammates can develop in whatever Python environment they like.

## Bonus step: Running in CI

If you use Pre-commit locally it's often a good idea to run `pre-commit run -a` in your CI pipeline. The setup I gave at the top of this post works fine in CI too. However, if you'd rather have a different Mypy setup in CI than locally, you can run Pre-commit with a SKIP environment variable in CI to skip the Mypy hook, and then run Mypy however you want in a separate CI job: `SKIP=mypy pre-commit run -a`. See [Pre-commit - Temporarily disabling hooks](https://pre-commit.com/#temporarily-disabling-hooks).

## Summary
We have seen many potential issues of running Mypy in Pre-commit:
- Changing one file may cause a type error in another file, so we need to run Mypy on all files, not just those that have changed
- We need to give Mypy access to the installed dependencies of the code it is type-checking, otherwise it can't check the usages of those dependencies
- Flags that reduce the strictness of Mypy like `--ignore-missing-imports` can give us false confidence 

We saw how to address these issues by making our own custom hook. There doesn't appear to be a neat, one-size-fits-all solution, so it's worth giving some thought to this set up in each instance.

