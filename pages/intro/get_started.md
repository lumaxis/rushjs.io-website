---
layout: page
title: Getting started
navigation_source: docs_nav
---

## 3 minute demo

Want to see Rush in action?  The only prerequisite you need is [NodeJS](https://github.com/nodejs/node).

**From your shell, install Rush like this:**
```sh
$ npm install -g @microsoft/rush
```

(Don't type the **"$"** of course.)  :-)

**For command-line help, do this:**
```sh
$ rush -h
```

**To see Rush build some real projects, try running these commands:**
```sh
$ git clone https://github.com/microsoft/rushstack
$ cd rushstack

# Install the npm packages:
# (If you don't have a GitHub email configured, add the "--bypass-policy" option.)
$ rush update

# Incremental install:
$ rush update  # <-- instantaneous!

# Force all projects to be rebuilt:
$ rush rebuild

# Incremental build:
$ rush build    # <-- instantaneous!

# Use "--verbose" to view the console logs for each project as it is built.
# Projects build in parallel processes, but their logs are collated.
$ rush rebuild --verbose
```


## Let's get started!

Choose your tutorial scenario...

- [I'm a developer.]({% link pages/developer/new_developer.md %}) Learn how to work in a repo that already uses Rush.
- [I'm a repo maintainer.]({% link pages/maintainer/setup_new_repo.md %})  Learn how to convert your repo to use the Rush system.
