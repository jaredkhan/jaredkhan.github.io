---
layout: post
title:  "Running Mypy in Pre-commit"
date:   2020-12-14 22:05:00 +0000
tags: Git Python
---

The only thing worse than not type-checking your code is thinking you are type-checking it when you aren't. 

This post is about running Mypy in a pre-commit hook using the [Pre-commit](https://pre-commit.com/) framework.  Running Mypy is a little fiddly in itself, and [pre-commit/mirrors-mypy](https://github.com/pre-commit/mirrors-mypy) (the de facto way to call Mypy in Pre-commit) calls Mypy in a slightly opinionated way that may introduce more confusions or hide errors you want to see.

Here, I show you how to make your own Mypy hook that suits your needs. The main issues it covers are:

- Making sure you run Mypy on all files, not just those that have changed
- Making sure Mypy has access to the installed dependencies of the code it is type-checking
- Be careful with the use of flags that reduce the strictness of Mypy like `--ignore-missing-imports`

## A solution that works in my case

Before discussing the gory details and alternatives, here's a solution that works for my project.

I add a `mypy.ini`:

```bash
[mypy]
# This will vary (and may not be necessary) for your project layout
mypy_path=./src:./tests

# Explicitly blacklist modules in use that don't have type stubs
[mypy-pytest.*]
ignore_missing_imports = True

[mypy-pyproj.*]
ignore_missing_imports = True
```

and then add a script at `./run-mypy`:

```bash
#!/usr/bin/env bash

# A script for running mypy, with all its dependencies installed.

set -o errexit

# Install the dependencies into the mypy environment.
# Avoid pip spending a lot of time quietly retrying 
# since most likely cause of failure is lack of VPN connection.
pip install --editable . \
  --index-url https://my-custom-index-url.com/pypi/simple \
  --retries 1 \
  --no-input \
  --quiet

# Ignores the list of changed files that pre-commit passes to this script,
# as this may miss type errors.
# Note that we do not use --ignore-missing-imports, 
# as this can give us false confidence in our results.
mypy --package acme --namespace-packages
```

and then run that script in the Pre-commit config: `./.pre-commit-config.yaml`

```bash
- repo: local
    # We do not use pre-commit/mirrors-mypy, as it comes with opinionated defaults (--ignore-missing-imports)
    # and is difficult to configure to run with the dependencies correctly installed
    hooks:
      - id: mypy
        name: mypy
        entry: "./run-mypy"
        language: python
        language_version: python3.7
        additional_dependencies: ["mypy==0.790"]
        always_run: true
        types: [python]
				# use require_serial so that the script is only called once per commit
        require_serial: true
        # Leave this in verbose mode as it's easy to mis-configure Mypy
        # so reassuring to see how many files it has run on.
        verbose: true
```

To expose all the issues this tries to cover, we'll build it up in 3 steps.

## Step 1: Running Mypy correctly outside of Pre-commit

Running `mypy .` in the root of your project unfortunately will often **not** do exactly what you need it to do. You will likely need to play around to make sure Mypy is running on all the files that you want and that it has correctly figured out their full module names. This involves choosing:

- Whether to specify the files to type-check as a package, a module, a directory, or a file path
- Whether to specify a MYPYPATH
- Whether to add the `--namespace-packages` or `--python-executable` options
- What working directory to invoke Mypy from

[Running mypy and managing imports](https://mypy.readthedocs.io/en/stable/running_mypy.html#running-mypy-and-managing-imports) is a helpful section of the documentation for getting this right. Pay extra attention when you are using namespace packages, packages without `__init__.py` files. 

I'm writing this whilst v0.790 is the latest release. Simplifying the calling of Mypy, and its import handling is a current priority for the maintainers. See for example the umbrella issue, [#8584 — Redesign import handling](https://github.com/python/mypy/issues/8584). Various improvements have already been merged to the master branch.

Once Mypy is running on the correct files,  you'll want to get it running the right checks for your codebase so that it passes whilst also checking what you want it to check. This may involve:

- Making changes to your codebase to meet new rules that you want to enforce
- Setting various strictness settings. For example: `--no-implicit-optional`, `--disallow-untyped-defs`, `-no-strict-optional` or the umbrella option `--strict`
- Deciding which imports to ignore. Sometimes Mypy will complain that it can't find a certain module or its stubs. This can be indicative that the way you are running Mypy has not given it access to these dependencies, which you should fix (see below), but can also mean that the library doesn't have any type stubs. For the latter case, it is sensible to treat those modules as `Any` by adding them to a `mypy.ini` file.

```bash
# mypy.ini

[mypy]
# this section is required
# you can also add a mypy_path here, if you need one

[mypy-pyproj.*]
# ignore missing stubs for pyproj and its subpackages.
# it is not annotated
ignore_missing_imports = True

```

### Bake it into a script

Now that you know precisely how you want to call Mypy, create a script called `run-mypy` that captures the arguments you want to use. For example, in my case, I have a few packages in the `acme` namespace in the `src/acme` directory, and my script ended up looking like this:

```bash
#!/usr/bin/env bash

# Note the use of --package rather than using the path src/acme,
# which would correctly collect our files
# but erroneously add src/acme to the path.
# We only want src in the path so that our modules are known to Mypy by their fully qualified names.
# Note the use of --namespace-packages which allows Mypy to find our packages despite their being namespace packages.
mypy --package acme --namespace-packages
```

## Step 2: Creating our own Pre-commit hook

### How Pre-commit runs hooks

Pre-commit installs each Python hook in a separate virtualenv. Before each commit, the list of staged files is passed to that hook (any unstaged changes are stashed and restored after all hooks have run).

**Problem: Only running changed files**

With Mypy, we probably don't want to pass it just the list of changed files:

- It will miss type errors resulting from but not occurring in the staged changes. For example: if you have changed the definition of a function but not a usage of that function in another file then the usage is now invalid, but won't be checked.
- As mentioned above, you may need more control over how Mypy is invoked anyway.
- Running Mypy in full is probably not a problem. Mypy uses an [Incremental Mode](https://mypy.readthedocs.io/en/stable/command_line.html#incremental-mode) by default. It stores calculated type information so re-running it after a few changes doesn't take as long. For faster incremental runs, consider using a long-running [Mypy daemon](https://mypy.readthedocs.io/en/stable/mypy_daemon.html#mypy-daemon).

We'll solve this by using our own `run-mypy` script and ignoring the file list that Pre-commit passes to it.

**Problem: Running in an isolated virtualenv**

Mypy running in a separate virtualenv is also problematic, since it won't have access to all of the dependencies installed in your main development environment. This means it can't type check usages of any of those dependencies.

### Setting up the hook

We can solve both these problems with a properly-configured hook, which we'll set up ourselves. To get started, create a new [Repository-local hook](https://pre-commit.com/#repository-local-hooks) by adding the following to your .pre-commit-config.yaml like so

```bash
- repo: local
    hooks:
      - id: mypy
        name: mypy
        entry: "./run-mypy"
        language: python
        language_version: python3.7  # Replace with appropriate version
        additional_dependencies: ["mypy==0.790"]
        types: [python]
				# use require_serial so that the script is only called once per commit
        require_serial: true
        # Leave this in verbose mode as it's easy to mis-configure Mypy
        # so reassuring to see how many files it has run on.
        verbose: true
```

## Step 3: Giving the Mypy hook access to dependencies

Mypy needs an environment where the dependencies are imported so that it can check for type-errors in their usage. Here's a few options for that:

### Option 1: Run Mypy in the environment that you already have setup with `language: system`

Replace `language: python` in your hook definition with `language: system`. This will stop Pre-commit creating a separate virtualenv for the hook and just run it in whatever environment you happen to be in when you run `git commit` or `pre-commit run`. This means you always run Mypy directly in your dev environment, but breaks if any of the developers on the project want to trigger Pre-commit from outside the dev environment. For example, this won't work if using a GUI Git client, as the correct virtualenv probably won't be activated.

### Option 2: Point Mypy to the environment with `--python-executable`

If it's possible to automatically figure out the path to the appropriate Python interpreter (the one associated with the existing installation of your dependencies, which may or may not be in a virtual environment), then you can point Mypy to that path using the `--python-executable` option on `mypy`.

### Option 3: Install specific dependencies with the `additional_dependencies` hook option

If you only care about type-checking the usages of a few third-party modules, then you can install those specific modules into the hook environment like so:

```bash
- repo: local
    hooks:
      - id: mypy
        name: mypy
        entry: "./run-mypy"
        language: python
        language_version: python3.7  # Replace with appropriate version
        additional_dependencies: ["mypy==0.790"]
        types: [python]
				# use require_serial so that the script is only called once per commit
        require_serial: true
        # Leave this in verbose mode as it's easy to mis-configure Mypy
        # so reassuring to see how many files it has run on.
        verbose: true
				additional_dependencies: [""]
```

This is relatively fast as Pre-commit remembers the dependencies it installed in the environment [[source code]](https://github.com/pre-commit/pre-commit/blob/cf604f6b93b8fb7a61ab9ca45e5dedbdb4fd5796/pre_commit/repository.py#L54). The downside is this means duplicating your list of dependencies (at least those that have type stubs).

The additional_dependencies are just sent directly to Pip [[source code]](https://github.com/pre-commit/pre-commit/blob/cf604f6b93b8fb7a61ab9ca45e5dedbdb4fd5796/pre_commit/languages/python.py#L199) so you can happily add, for example, a `--index-url` argument in this array. Just be aware that Pre-commit will only re-run Pip when the list of `additional_dependencies` changes, so don't expect to put "requirements.txt" in this array and have it figure out when you've changed that file.

### Option 4: Running a full `pip install` in the hook

This is not fast. The speed we mostly care about is that of running the hook on each commit, not its initial setup. However, running `pip install` takes many seconds even when the dependencies are already installed. It is, however, a pretty reliable and easy way to make sure your dependencies are installed if the performance hit is acceptable to you and your team.

To do this, simply add the appropriate `pip` command into your `run-mypy` script. For example:

```bash
#!/usr/bin/env bash

# ...

set -o errexit

pip install --editable . \
  --no-input \
  --quiet

# ...
mypy --package acme --namespace-packages
```

## Bonus step: Running in CI

If you use Pre-commit locally it's often a good idea to run `pre-commit run -a` in your CI pipeline. The setup I gave at the top of this post works fine in CI too. However, if you'd rather have a different Mypy setup in CI than locally, you can run Pre-commit with a SKIP environment variable in CI to skip the Mypy hook, and then run Mypy however you want in a separate CI job: `SKIP=mypy pre-commit run -a`. See [Pre-commit - Temporarily disabling hooks](https://pre-commit.com/#temporarily-disabling-hooks).