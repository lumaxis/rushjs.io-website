---
layout: page
title: npm vs pnpm vs Yarn
navigation_source: docs_nav
---

Before you can start installing a JavaScript library, you need to choose which package manager you will use.  (Our community loves flexibility and choices, so of course there's not just one!)  Rush supports the three most popular package managers.  In chronological order:

- [npm](https://docs.npmjs.com/getting-started/what-is-npm): the tool that pioneered the packaging standard and registry protocol used by most JavaScript package managers today.  The tool's developers also maintain the npmjs.com registry, which is currently the most popular place to distribute open source JavaScript libraries.

- [Yarn](https://yarnpkg.com/en/): a complete rewrite of the npm tool that preserves the same installation model, but promises faster installations, better reliability, and some cool new features (e.g. Yarn workspaces) that facilitate large scale development.

- [pnpm](https://pnpm.js.org/): A fundamentally new installation model that solves the ["phantom dependency" and "npm doppelganger"]({% link pages/advanced/phantom_deps.md %})" problems, while cleverly making use of [symlinks](https://en.wikipedia.org/wiki/Symbolic_link) to remain 100% compatible with the NodeJS module resolution standard.


## Which one should I use with Rush?

The answer depends on your needs.  The Rush developers don't endorse a particular package manager, but here are some observations based on our experience from managing our own monorepos:

#### Considerations for npm

- npm is the most compatible choice, and the most forgiving for dealing with "bad" packages.

- If you choose npm, you may need to use an older release.  npm 5.x and 6.x are both known to have unresolved regressions that cause trouble in Rush repos.  npm **4.5.0** is the most recent version that's known to work very reliably, but unfortunately it's pretty old.  (We'd greatly appreciate community help improving this situation. We're using [GitHub issue #886](https://github.com/microsoft/rushstack/issues/886) to track this effort.)

  *Before reporting a Rush bug involving the npm package manager, first try downgrading to `"npmVersion": "4.5.0"`.  If that eliminates the repro, then your issue is likely an npm regression and may not be fixable in the Rush code base.  We still accept these issues, but we track them differently.*

#### Considerations for pnpm

- pnpm is the only option that solves the [npm doppelgangers]({% link pages/advanced/npm_doppelgangers.md %}) problem.  In a complex monorepo, doppelgangers sometimes cause a lot of trouble, so pnpm has an important advantage in this regard.

- Although pnpm's symlinking strategy correctly follows the modern NodeJS module resolution standard, many legacy packages do not, which causes compatibility problems.  Teams who migrate existing projects from Yarn/npm to pnpm often encounter "bad packages" that need workarounds or fixes.  The incompatibilities generally reflect real problems with those packages: (1) forgetting to list dependencies in the **package.json** file, or (2) implementing homebrew module resolution without handling symlinks according to the standard.  Most "bad" packages have straightforward fixes, but it may seem daunting for a small team.  (The [pnpm Discord chat room](https://discord.gg/mThkzAT) is a great resource for help, though.)

- pnpm is newer and less widely used than npm or Yarn, but it's a solid piece of software.  Microsoft uses pnpm in Rush repos with hundreds of projects and hundreds of PRs per day, and we've found it to be very fast and reliable.

- pnpm is currently the only option that supports the `--strict-peer-dependencies` protection (see `"strictPeerDependencies"` in **rush.json**).

#### Considerations for Yarn

- Rush's support for Yarn is relatively new and unproven, so we're eager to hear about issues and get them fixed.

- Yarn installs faster than npm (although somewhat slower than pnpm).

- Yarn's "resolutions" feature is not yet compatible with Rush.  (See [Rush issue #831](https://github.com/microsoft/rushstack/issues/831).)

- Yarn's "workspaces" are not used in a Rush repo, since they rely on an installation model that doesn't protect against phantom dependencies.  Rush's linking strategy is mostly equivalent to workspaces, however.

## Specifying your package manager

To change your package manager, edit the **rush.json** file and uncomment one of the three fields (`npmVersion`, `pnpmVersion`, or `yarnVersion`):

**rush.json**
```
/**
  * The next field selects which package manager should be installed and determines its version.
  * Rush installs its own local copy of the package manager to ensure that your build process
  * is fully isolated from whatever tools are present in the local environment.
  *
  * Specify one of: "pnpmVersion", "npmVersion", or "yarnVersion".  See the Rush documentation
  * for details about these alternatives.
  */
"pnpmVersion": "2.15.1",

// "npmVersion": "4.5.0",
// "yarnVersion": "1.9.4",
```

After changing the setting, delete your old shrinkwrap file and other package manager specific files from the **common/config/rush** folder.  (Otherwise Rush will complain about unsupported config files.)  Then run `rush update --full --purge`.  That's it!
