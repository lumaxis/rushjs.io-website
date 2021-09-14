---
layout: page
title: npm registry authentication
navigation_source: docs_nav
---

A **private npm registry** enables your monorepo to publish npm packages for internal usage.  It works the same as
the public [https://www.npmjs.com/](https://www.npmjs.com/) registry, except that accessing the private registry
requires authorization.  Each user will need to obtain an access token that typically gets stored in the
[~/.npmrc file](https://docs.npmjs.com/cli/v6/configuring-npm/npmrc)
on their computer.

Most large monorepos eventually require a private npm registry.  It's useful for:

- sharing code privately between teams
- proxying access to the public registry, to improve reliability, audit package usage, and apply security screening
- speeding up CI operations by installing prebuilt tooling packages, instead of performing `rush install && rush build`
  before a tool can be invoked
- testing installation behaviors before publishing a package to public npm registry
- publishing wrappers or temporary forks of third-party packages<br/>
  (Compared to [GitHub URL dependencies](https://docs.npmjs.com/cli/v7/configuring-npm/package-json#github-urls),
  npm packages give you proper SemVer versioning and better caching semantics.)

Some popular providers are:

- [AWS CodeArtifact](https://aws.amazon.com/blogs/devops/publishing-private-npm-packages-aws-codeartifact/)
- [Azure DevOps Artifacts](https://docs.microsoft.com/en-us/azure/devops/artifacts/get-started-npm?view=azure-devops)
- [GitHub Packages](https://github.com/features/packages)
- [GitLab Package Registry](https://docs.gitlab.com/ee/user/packages/npm_registry/)
- [JFrog Artifactory](https://jfrog.com/artifactory/)
- [npm private packages](https://docs.npmjs.com/about-private-packages)

And for testing purposes, [Verdaccio](https://verdaccio.org/) is a lightweight Node.js server that can run
on `http://localhost` and implements a complete private registry with proxy capabilities.


## Registry mappings

The mappings for your private registry are specified in
[the monorepo .npmrc file]({% link pages/configs/npmrc.md %}).

Below is an example configuration that installs company packages from the private
registry, but gets all other packages from the public registry.  The company packages are
identified by their `@example` npm scope.

**common/config/rush/.npmrc**
```shell
# Map your company's npm scope ("@example") to the private registry URL:
@example:registry=https://my-registry.example.com/npm-private/

# Otherwise, all other packages come from the public npm registry:
registry=https://registry.npmjs.org/
always-auth=false

# Here we specify how the package manager should authenticate to the private registry.
# For security reasons, CI jobs should obtain their tokens from environment variables.
# The exact syntax depends on your registry provider.  If a line references an environment
# variable that is undefined, Rush will ignore that line.  This avoids producing an invalid
# string that might interfere with a developer who obtains their credentials from ~/.npmrc.
//my-registry.example.com/npm-private/:_password=${MY_CI_TOKEN}
//my-registry.example.com/npm-private/:username=${MY_CI_USER}
//my-registry.example.com/npm-private/:always-auth=true
```

More commonly, your private registry will act as a **caching proxy** so that it can provide
packages from the public npm registry as well.  In this case, npm scopes don't need to be mapped.
Your setup might look like this:

**common/config/rush/.npmrc**
```shell
# Map everything to the private registry URL
registry=https://my-registry.example.com/npm-private/
always-auth=true

# Here we specify how the package manager should authenticate to the private registry.
# For security reasons, CI jobs should obtain their tokens from environment variables.
# The exact syntax depends on your registry provider.  If a line references an environment
# variable that is undefined, Rush will ignore that line.  This avoids producing an invalid
# string that might interfere with a developer who obtains their credentials from ~/.npmrc.
//my-registry.example.com/npm-private/:_password=${MY_CI_TOKEN}
//my-registry.example.com/npm-private/:username=${MY_CI_USER}
```

> For details about the lookup precedence for **.npmrc** settings,
see the [.npmrc]({% link pages/configs/npmrc.md %}) page.


## Prompting for credentials with "rush setup"

Rush recently introduced an experimental feature where `rush install` can detect when a user's registry credentials
are missing or expired.  If so, they are asked to run `rush setup`, which walks the user through the process of
obtaining a token, and then updates their **~/.npmrc** file.  The new settings will be intelligently merged with
any existing contents of that file.

A sample `rush setup` interaction looks like this:
```
npm credentials are missing or expired

==> Fix this problem now? (y/N) Yes

This monorepo consumes packages from an Artifactory private npm registry.

==> Do you already have an Artifactory user account? (y/n) Yes

Please open this URL in your web browser:

  https://my-company.jfrog.io/

Your user name appears in the upper-right corner of the JFrog website.

==> What is your Artifactory user name? example-user

Click "Edit Profile" on the JFrog website.  Click the "Generate API Key" button if you haven't already done so
previously.

==> What is your Artifactory API key? ***************

Fetching an npm token from the Artifactory service...

Adding Artifactory token to: /home/example-user/.npmrc
```

The initial implementation supports the [JFrog Artifactory](https://jfrog.com/artifactory/) service only.
Other services will be implemented in the future.

To use this feature, simply assign the `"registryUrl"` field and set `"enabled": true` in your
[artifactory.json]({% link pages/configs/artifactory_json.md %}) config file.
The file template contains documentation for other optional settings that can be used to customize
the dialogue.


## See also

- [rush setup]({% link pages/commands/rush_setup.md %})
- [artifactory.json]({% link pages/configs/artifactory_json.md %}) config file
- [.npmrc]({% link pages/configs/npmrc.md %}) config file
- [.npmrc-publish]({% link pages/configs/npmrc-publish.md %}) config file
