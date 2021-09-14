---
layout: page
title: Enabling the build cache (experimental)
navigation_source: docs_nav
---

Rush has always supported an [incremental analyzer]({% link pages/advanced/incremental_builds.md %}) that
enables `rush build` to skip projects whose input files have not changed since the last build.  (This optimization
can also be used with custom commands by enabling the `incremental` flag in **custom-commands.json**.)  However,
the build output is not saved anywhere, so generally a full rebuild is still required when switching to another branch.

Rush's experimental new **build cache** improves on this by creating a tar archive of each project's build outputs.
The archive is cached so that later, if `rush build` can find a match in the cache, it can extract the archive
instead of building that project.  This can provide dramatic speedups, for example reducing a 30 minute build time
to 30 seconds.  The cache key is a hash of the source files and npm dependencies, following the
[same basic rules]({% link pages/advanced/incremental_builds.md %}) as the incremental analyzer.

The build cache archives are stored in two places:

- **In a cache folder on your local disk.**  This way you can switch between different branches without
  losing your incremental build state.  You can even configure a centralized folder to be shared between
  multiple enlistments on your machine.  The default location is **common/temp/build-cache**.

- **In a cloud-hosted storage container. (Optional)**  In a typical setup, the CI system would be configured to write
  to cloud storage, and individual users are granted read-only access.  For example, each time a PR is merged into
  the `master` branch, the CI system builds that baseline and uploads it to cloud storage.  Even for a user who
  is doing `git clone` for the first time, their `rush build` will be very fast.

> Build caching is considered a replacement for build skipping, so once enabled, commands that support
> incremental building will begin saving and restoring from the cache instead of the previous "skipping"
> behavior. Projects that haven't been configured for build caching, or intentionally disable build
> caching, will continue to use the default build skipping behavior.

## Enabling the local disk cache

The build cache feature is enabled using the [build-cache.json]({% link pages/configs/build-cache_json.md %})
config file.  You can copy the template from the website or use `rush init` to create this file.

To enable the basic local disk cache, add these two settings:

**common/config/rush/build-cache.json**
```js
{
  . . .
  /**
   * (Required) EXPERIMENTAL - Set this to true to enable the build cache feature.
   *
   * See https://rushjs.io/pages/maintainer/build_cache/ for details about this experimental feature.
   */
  "buildCacheEnabled": true,

  /**
   * (Required) Choose where project build outputs will be cached.
   *
   * Possible values: "local-only", "azure-blob-storage", "amazon-s3"
   */
  "cacheProvider": "local-only",

  . . .
}
```

> **Upgrade note:** Early releases of this feature were enabled using the `"buildCache": true` setting
> in **experiments.json**. This has been superseded by `"buildCacheEnabled"` in **build-cache.json**.


## Configuring project output folders

With only this change, if you run `rush rebuild --verbose`, you will see this warning:

```
Project does not have a rush-project.json configuration file, or one provided by a rig,
so it does not support caching.
```

The build cache needs to know which folders should be stored in the tar archive.  Those details vary between
toolchains, and are thus configured separately for each project using the
[rush-project.json]({% link pages/configs/rush-project_json.md %}) config file.

For example:

**&lt;your-project&gt;/config/rush-project.json**

```js
{
  . . .

  /**
   * Specify the folders where your toolchain writes its output files.  If enabled, the Rush build cache will
   * restore these folders from the cache.
   *
   * The strings are folder names under the project root folder.  These folders should not be tracked by Git.
   * They must not contain symlinks.
   */
  "projectOutputFolderNames": ["lib", "dist"]
  . . .
}
```

It's recommended to use a [rig package](https://rushstack.io/pages/heft/rig_packages/) to avoid having
to copy this file into each project folder.


## Testing the build cache

Now you should see projects being cached as shown in this sample log output:

```shell
$ rush rebuild --verbose

. . .

==[ example-project ]==============================================[ 1 of 5 ]==

This project was not found in the build cache.

Invoking: heft test --clean

. . .

Caching build output folders: lib

Successfully set cache entry.

"example-project" completed successfully in 11.27 seconds.
```

When we run the same command a second time, Rush extracts the archive instead of invoking the build task:

```shell
$ rush rebuild --verbose

. . .

==[ example-project ]==============================================[ 1 of 5 ]==

Build cache hit.

Clearing cached folders: lib, dist

Successfully restored output from the build cache.

example-project was restored from the build cache.
```

Note that `rush rebuild` will not read from cache. To disable writing from cache during `rush rebuild`, set the
[`RUSH_BUILD_CACHE_WRITE_ALLOWED`]({% link pages/configs/environment_vars.md %}) environment variable to `0`.

By default, the cached tar archives are stored under your **common/temp/build-cache** folder
(and thus will be cleaned by `rush purge`).  It is safe to delete these files.


## Enabling cloud storage

Currently the `cacheProvider` setting provides three choices:

- `"local-only"`: no cloud storage; archives are only kept on a local disk folder
- `"azure-blob-storage"`: Microsoft Azure [blob storage container](https://docs.microsoft.com/en-us/azure/storage/blobs/)
- `"amazon-s3"`: Amazon [S3 bucket](https://docs.aws.amazon.com/AmazonS3/latest/userguide/UsingBucket.html)

Other options may be implemented in the future.  (Consider contributing one by implementing a subclass of
[CloudBuildCacheProviderBase](https://github.com/microsoft/rushstack/blob/master/apps/rush-lib/src/logic/buildCache/CloudBuildCacheProviderBase.ts).)

For example, here's how we would configure an Azure blob container:

**common/config/rush/build-cache.json**
```js
{
  . . .
  /**
   * (Required) EXPERIMENTAL - Set this to true to enable the build cache feature.
   *
   * See https://rushjs.io/pages/maintainer/build_cache/ for details about this experimental feature.
   */
  "buildCacheEnabled": true,

  /**
   * (Required) Choose where project build outputs will be cached.
   *
   * Possible values: "local-only", "azure-blob-storage", "amazon-s3"
   */
  "cacheProvider": "azure-blob-storage",

  /**
   * Use this configuration with "cacheProvider"="azure-blob-storage"
   */
  "azureBlobStorageConfiguration": {
    /**
     * (Required) The name of the the Azure storage account to use for build cache.
     */
    "storageAccountName": "example",

    /**
     * The name of the container in the Azure storage account to use for build cache.
     */
    "storageContainerName": "my-container"

    /**
     * If set to true, allow writing to the cache. Defaults to false.
     */
    "isCacheWriteAllowed": false

  . . .
```

Note that we have set `"isCacheWriteAllowed": false` to prevent regular users from writing to the container.
(Later, we will use an environment variable to override this for our CI job.)


## User authentication

If security is not a priority for your repo, you can simplify user setup by configuring your storage container
to allow unauthenticated anonymous access.  The container is accessed via an HTTPS URL containing randomized
hashes which are difficult to guess without access to your Git repo.  This provides rudimentary
[security through obscurity](https://en.wikipedia.org/wiki/Security_through_obscurity).

A more security-conscious organization however will prefer to require authentication even for read-only access.
Rush provides a [rush update-cloud-credentials]({% link pages/commands/rush_update-cloud-credentials.md %})
command to make this easy for users to set up:

```shell
$ rush update-cloud-credentials --interactive


Rush Multi-Project Build Tool 5.45.6 (unmanaged) - https://rushjs.io
Node.js version is 12.20.1 (LTS)


Starting "rush update-cloud-credentials"

 ╔═════════════════════════════════════════════════════════════════════════╗
 ║             To sign in, use a web browser to open the page              ║
 ║     https://microsoft.com/devicelogin and enter the code XAYBQEGRK      ║
 ║                            to authenticate.                             ║
 ╚═════════════════════════════════════════════════════════════════════════╝
```

The credentials are stored in the user's home directory under `~/.rush-user/credentials.json`.


## CI setup

In a typical configuration, users have read-only access and the cache is populated by an automation account;
for example, a CI job that builds your `master` branch after each PR is merged.  In our example above, the
`"isCacheWriteAllowed": false` setting is what prevents users from writing to the cache.  The CI job can
override this by setting the [RUSH_BUILD_CACHE_WRITE_ALLOWED]({% link pages/configs/environment_vars.md %})
environment variable, and by providing credentials for the CI environment in the
[RUSH_BUILD_CACHE_CREDENTIAL]({% link pages/configs/environment_vars.md %}) environment variable.

For Azure Blob Storage, this must be a SAS token serialized as query parameters.
See [this article](https://docs.microsoft.com/en-us/azure/storage/common/storage-sas-overview) for details
about SAS tokens.  You can obtain a SAS token via the [Settings > Access keys](
https://docs.microsoft.com/en-us/azure/storage/common/storage-account-keys-manage?tabs=azure-portal)
page for your storage account.

If your CI system uses a custom build orchestrator with Rush
(for example [BuildXL](https://github.com/Microsoft/BuildXL)),
the [rush write-build-cache]({% link pages/commands/rush_write-build-cache.md %}) command enables you to
populate a cache entry from a specific project's output folders.

> The build cache feature is still under development.  Feedback is welcome!
>
> Some relevant GitHub issues to follow:
> - [Build cache feature #2393](https://github.com/microsoft/rushstack/issues/2393) - the original feature spec
> - [Build Cache: split apart RUSH_BUILD_CACHE_WRITE_CREDENTIAL #2642](https://github.com/microsoft/rushstack/issues/2642)
> - [Allow project config to specify non-build-related files #2618](https://github.com/microsoft/rushstack/issues/2618)
> - ["tar" exited with code 1 while attempting to create the cache entry #2622](https://github.com/microsoft/rushstack/issues/2622)
