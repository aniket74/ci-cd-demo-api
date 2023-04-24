Readme file. Added build.yml in github actions..

About Build.yaml file -

1. The Workflow is triggered by a push on the main branch.
2. For other branches in GitHub and for pull request trigger, add the branch names.

This workflow will first  build the MuleSoft project and deploy it to CloudHub. This workflow contains two jobs:
1. To build the application.
2. To deploy the application to CloudHub. 

Build Job - 
1. The Java environment for building the application would be set up because MuleSoft is based on java.
2. In the next step packaging will create an output jar file which will be used to deploy the application.
3. Every artifact will be stamped with the commit hash. So, we can find which commit is being deployed. 
4. In the last step of the build job, the build artifact will be uploaded. Which will allow the build and deploy job to share the data between them.
5. You can find the artifact file in your repository in the Actions tab after the job is completed.

Deploy Job - 
1.The deploy job will first check the repository and will retrieve all the related dependencies.
2. In the second step it will download the artifact file uploaded by the build job.
3. In the third and final it will retrieve the Anypoint platform’s username and password from the Secrets to deploy the application to the CloudHub. 

Deploy Job will only execute if the Build job is successfully completed.

------------------------------------------------------------------------------------------------------------------
# Github Actions Cache --->
# Cache action

This action allows caching dependencies and build outputs to improve workflow execution time.

>Two other actions are available in addition to the primary `cache` action:
>* [Restore action](./restore/README.md)
>* [Save action](./save/README.md)

[![Tests](https://github.com/actions/cache/actions/workflows/workflow.yml/badge.svg)](https://github.com/actions/cache/actions/workflows/workflow.yml)

## Documentation

See ["Caching dependencies to speed up workflows"](https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows).

## What's New

### v3

* Added support for caching in GHES 3.5+.
* Fixed download issue for files > 2GB during restore.
* Updated the minimum runner version support from node 12 -> node 16.
* Fixed avoiding empty cache save when no files are available for caching.
* Fixed tar creation error while trying to create tar with path as `~/` home folder on `ubuntu-latest`.
* Fixed zstd failing on amazon linux 2.0 runners.
* Fixed cache not working with github workspace directory or current directory.
* Fixed the download stuck problem by introducing a timeout of 1 hour for cache downloads.
* Fix zstd not working for windows on gnu tar in issues.
* Allowing users to provide a custom timeout as input for aborting download of a cache segment using an environment variable `SEGMENT_DOWNLOAD_TIMEOUT_MINS`. Default is 10 minutes.
* New actions are available for granular control over caches - [restore](restore/action.yml) and [save](save/action.yml).
* Support cross-os caching as an opt-in feature. See [Cross OS caching](./tips-and-workarounds.md#cross-os-cache) for more info.
* Added option to fail job on cache miss. See [Exit workflow on cache miss](./restore/README.md#exit-workflow-on-cache-miss) for more info.
* Fix zstd not being used after zstd version upgrade to 1.5.4 on hosted runners
* Added option to lookup cache without downloading it.
* Reduced segment size to 128MB and segment timeout to 10 minutes to fail fast in case the cache download is stuck.

See the [v2 README.md](https://github.com/actions/cache/blob/v2/README.md) for older updates.

## Usage

### Pre-requisites

Create a workflow `.yml` file in your repository's `.github/workflows` directory. An [example workflow](#example-cache-workflow) is available below. For more information, see the GitHub Help Documentation for [Creating a workflow file](https://help.github.com/en/articles/configuring-a-workflow#creating-a-workflow-file).

If you are using this inside a container, a POSIX-compliant `tar` needs to be included and accessible from the execution path.

If you are using a `self-hosted` Windows runner, `GNU tar` and `zstd` are required for [Cross-OS caching](https://github.com/actions/cache/blob/main/tips-and-workarounds.md#cross-os-cache) to work. They are also recommended to be installed in general so the performance is on par with `hosted` Windows runners.

### Inputs

* `key` - An explicit key for a cache entry. See [creating a cache key](#creating-a-cache-key).
* `path` - A list of files, directories, and wildcard patterns to cache and restore. See [`@actions/glob`](https://github.com/actions/toolkit/tree/main/packages/glob) for supported patterns.
* `restore-keys` - An ordered list of prefix-matched keys to use for restoring stale cache if no cache hit occurred for key.
* `enableCrossOsArchive` - An optional boolean when enabled, allows Windows runners to save or restore caches that can be restored or saved respectively on other platforms. Default: `false`
* `fail-on-cache-miss` - Fail the workflow if cache entry is not found. Default: `false`
* `lookup-only` - If true, only checks if cache entry exists and skips download. Does not change save cache behavior. Default: `false`

#### Environment Variables

* `SEGMENT_DOWNLOAD_TIMEOUT_MINS` - Segment download timeout (in minutes, default `10`) to abort download of the segment if not completed in the defined number of minutes. [Read more](https://github.com/actions/cache/blob/main/tips-and-workarounds.md#cache-segment-restore-timeout)

### Outputs

* `cache-hit` - A boolean value to indicate an exact match was found for the key.

    > **Note** `cache-hit` will only be set to `true` when a cache hit occurs for the exact `key` match. For a partial key match via `restore-keys` or a cache miss, it will be set to `false`.

See [Skipping steps based on cache-hit](#skipping-steps-based-on-cache-hit) for info on using this output

### Cache scopes

The cache is scoped to the key, [version](#cache-version), and branch. The default branch cache is available to other branches.

See [Matching a cache key](https://help.github.com/en/actions/configuring-and-managing-workflows/caching-dependencies-to-speed-up-workflows#matching-a-cache-key) for more info.

### Example cache workflow

#### Restoring and saving cache using a single action

```yaml
name: Caching Primes

on: push

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v3

    - name: Cache Primes
      id: cache-primes
      uses: actions/cache@v3
      with:
        path: prime-numbers
        key: ${{ runner.os }}-primes

    - name: Generate Prime Numbers
      if: steps.cache-primes.outputs.cache-hit != 'true'
      run: /generate-primes.sh -d prime-numbers

    - name: Use Prime Numbers
      run: /primes.sh -d prime-numbers
```

The `cache` action provides a `cache-hit` output which is set to `true` when the cache is restored using the primary `key` and `false` when the cache is restored using `restore-keys` or no cache is restored.

#### Using a combination of restore and save actions

```yaml
name: Caching Primes

on: push

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v3

    - name: Restore cached Primes
      id: cache-primes-restore
      uses: actions/cache/restore@v3
      with:
        path: |
          path/to/dependencies
          some/other/dependencies
        key: ${{ runner.os }}-primes
    .
    . //intermediate workflow steps
    .
    - name: Save Primes
      id: cache-primes-save
      uses: actions/cache/save@v3
      with:
        path: |
          path/to/dependencies
          some/other/dependencies
        key: ${{ steps.cache-primes-restore.outputs.cache-primary-key }}
```

> **Note**
> You must use the `cache` or `restore` action in your workflow before you need to use the files that might be restored from the cache. If the provided `key` matches an existing cache, a new cache is not created and if the provided `key` doesn't match an existing cache, a new cache is automatically created provided the job completes successfully.

## Caching Strategies

With the introduction of the `restore` and `save` actions, a lot of caching use cases can now be achieved. Please see the [caching strategies](./caching-strategies.md) document for understanding how you can use the actions strategically to achieve the desired goal.

## Implementation Examples

Every programming language and framework has its own way of caching.

See [Examples](examples.md) for a list of `actions/cache` implementations for use with:

* [C# - NuGet](./examples.md#c---nuget)
* [Clojure - Lein Deps](./examples.md#clojure---lein-deps)
* [D - DUB](./examples.md#d---dub)
* [Deno](./examples.md#deno)
* [Elixir - Mix](./examples.md#elixir---mix)
* [Go - Modules](./examples.md#go---modules)
* [Haskell - Cabal](./examples.md#haskell---cabal)
* [Haskell - Stack](./examples.md#haskell---stack)
* [Java - Gradle](./examples.md#java---gradle)
* [Java - Maven](./examples.md#java---maven)
* [Node - npm](./examples.md#node---npm)
* [Node - Lerna](./examples.md#node---lerna)
* [Node - Yarn](./examples.md#node---yarn)
* [OCaml/Reason - esy](./examples.md#ocamlreason---esy)
* [PHP - Composer](./examples.md#php---composer)
* [Python - pip](./examples.md#python---pip)
* [Python - pipenv](./examples.md#python---pipenv)
* [R - renv](./examples.md#r---renv)
* [Ruby - Bundler](./examples.md#ruby---bundler)
* [Rust - Cargo](./examples.md#rust---cargo)
* [Scala - SBT](./examples.md#scala---sbt)
* [Swift, Objective-C - Carthage](./examples.md#swift-objective-c---carthage)
* [Swift, Objective-C - CocoaPods](./examples.md#swift-objective-c---cocoapods)
* [Swift - Swift Package Manager](./examples.md#swift---swift-package-manager)
* [Swift - Mint](./examples.md#swift---mint)

## Creating a cache key

A cache key can include any of the contexts, functions, literals, and operators supported by GitHub Actions.

For example, using the [`hashFiles`](https://docs.github.com/en/actions/learn-github-actions/expressions#hashfiles) function allows you to create a new cache when dependencies change.

```yaml
  - uses: actions/cache@v3
    with:
      path: |
        path/to/dependencies
        some/other/dependencies
      key: ${{ runner.os }}-${{ hashFiles('**/lockfiles') }}
```

Additionally, you can use arbitrary command output in a cache key, such as a date or software version:

```yaml
  # http://man7.org/linux/man-pages/man1/date.1.html
  - name: Get Date
    id: get-date
    run: |
      echo "date=$(/bin/date -u "+%Y%m%d")" >> $GITHUB_OUTPUT
    shell: bash

  - uses: actions/cache@v3
    with:
      path: path/to/dependencies
      key: ${{ runner.os }}-${{ steps.get-date.outputs.date }}-${{ hashFiles('**/lockfiles') }}
```

See [Using contexts to create cache keys](https://help.github.com/en/actions/configuring-and-managing-workflows/caching-dependencies-to-speed-up-workflows#using-contexts-to-create-cache-keys)

## Cache Limits

A repository can have up to 10GB of caches. Once the 10GB limit is reached, older caches will be evicted based on when the cache was last accessed.  Caches that are not accessed within the last week will also be evicted.

## Skipping steps based on cache-hit

Using the `cache-hit` output, subsequent steps (such as install or build) can be skipped when a cache hit occurs on the key.  It is recommended to install missing/updated dependencies in case of a partial key match when the key is dependent on the `hash` of the package file.

Example:

```yaml
steps:
  - uses: actions/checkout@v3

  - uses: actions/cache@v3
    id: cache
    with:
      path: path/to/dependencies
      key: ${{ runner.os }}-${{ hashFiles('**/lockfiles') }}

  - name: Install Dependencies
    if: steps.cache.outputs.cache-hit != 'true'
    run: /install.sh
```

> **Note** The `id` defined in `actions/cache` must match the `id` in the `if` statement (i.e. `steps.[ID].outputs.cache-hit`)

## Cache Version

Cache version is a hash [generated](https://github.com/actions/toolkit/blob/500d0b42fee2552ae9eeb5933091fe2fbf14e72d/packages/cache/src/internal/cacheHttpClient.ts#L73-L90) for a combination of compression tool used (Gzip, Zstd, etc. based on the runner OS) and the `path` of directories being cached. If two caches have different versions, they are identified as unique caches while matching. This, for example, means that a cache created on a `windows-latest` runner can't be restored on `ubuntu-latest` as cache `Version`s are different.

> Pro tip: The [list caches](https://docs.github.com/en/rest/actions/cache#list-github-actions-caches-for-a-repository) API can be used to get the version of a cache. This can be helpful to troubleshoot cache miss due to version.

<details>
  <summary>Example</summary>
The workflow will create 3 unique caches with same keys. Ubuntu and windows runners will use different compression technique and hence create two different caches. And `build-linux` will create two different caches as the `paths` are different.

```yaml
jobs:
  build-linux:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Cache Primes
        id: cache-primes
        uses: actions/cache@v3
        with:
          path: prime-numbers
          key: primes

      - name: Generate Prime Numbers
        if: steps.cache-primes.outputs.cache-hit != 'true'
        run: ./generate-primes.sh -d prime-numbers

      - name: Cache Numbers
        id: cache-numbers
        uses: actions/cache@v3
        with:
          path: numbers
          key: primes

      - name: Generate Numbers
        if: steps.cache-numbers.outputs.cache-hit != 'true'
        run: ./generate-primes.sh -d numbers

  build-windows:
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@v3

      - name: Cache Primes
        id: cache-primes
        uses: actions/cache@v3
        with:
          path: prime-numbers
          key: primes

      - name: Generate Prime Numbers
        if: steps.cache-primes.outputs.cache-hit != 'true'
        run: ./generate-primes -d prime-numbers
```

</details>

## Known practices and workarounds

There are a number of community practices/workarounds to fulfill specific requirements. You may choose to use them if they suit your use case. Note these are not necessarily the only solution or even a recommended solution.

* [Cache segment restore timeout](./tips-and-workarounds.md#cache-segment-restore-timeout)
* [Update a cache](./tips-and-workarounds.md#update-a-cache)
* [Use cache across feature branches](./tips-and-workarounds.md#use-cache-across-feature-branches)
* [Cross OS cache](./tips-and-workarounds.md#cross-os-cache)
* [Force deletion of caches overriding default cache eviction policy](./tips-and-workarounds.md#force-deletion-of-caches-overriding-default-cache-eviction-policy)

### Windows environment variables

Please note that Windows environment variables (like `%LocalAppData%`) will NOT be expanded by this action. Instead, prefer using `~` in your paths which will expand to the HOME directory. For example, instead of `%LocalAppData%`, use `~\AppData\Local`. For a list of supported default environment variables, see the [Learn GitHub Actions: Variables](https://docs.github.com/en/actions/learn-github-actions/variables#default-environment-variables) page.
------------------------------------------------------------------------------------------------------------------

Github Actions Checkout --->

[![Build and Test](https://github.com/actions/checkout/actions/workflows/test.yml/badge.svg)](https://github.com/actions/checkout/actions/workflows/test.yml)

# Checkout V3

This action checks-out your repository under `$GITHUB_WORKSPACE`, so your workflow can access it.

Only a single commit is fetched by default, for the ref/SHA that triggered the workflow. Set `fetch-depth: 0` to fetch all history for all branches and tags. Refer [here](https://help.github.com/en/articles/events-that-trigger-workflows) to learn which commit `$GITHUB_SHA` points to for different events.

The auth token is persisted in the local git config. This enables your scripts to run authenticated git commands. The token is removed during post-job cleanup. Set `persist-credentials: false` to opt-out.

When Git 2.18 or higher is not in your PATH, falls back to the REST API to download the files.

# What's new

- Updated to the node16 runtime by default
  - This requires a minimum [Actions Runner](https://github.com/actions/runner/releases/tag/v2.285.0) version of v2.285.0 to run, which is by default available in GHES 3.4 or later.

# Usage

<!-- start usage -->
```yaml
- uses: actions/checkout@v3
  with:
    # Repository name with owner. For example, actions/checkout
    # Default: ${{ github.repository }}
    repository: ''

    # The branch, tag or SHA to checkout. When checking out the repository that
    # triggered a workflow, this defaults to the reference or SHA for that event.
    # Otherwise, uses the default branch.
    ref: ''

    # Personal access token (PAT) used to fetch the repository. The PAT is configured
    # with the local git config, which enables your scripts to run authenticated git
    # commands. The post-job step removes the PAT.
    #
    # We recommend using a service account with the least permissions necessary. Also
    # when generating a new PAT, select the least scopes necessary.
    #
    # [Learn more about creating and using encrypted secrets](https://help.github.com/en/actions/automating-your-workflow-with-github-actions/creating-and-using-encrypted-secrets)
    #
    # Default: ${{ github.token }}
    token: ''

    # SSH key used to fetch the repository. The SSH key is configured with the local
    # git config, which enables your scripts to run authenticated git commands. The
    # post-job step removes the SSH key.
    #
    # We recommend using a service account with the least permissions necessary.
    #
    # [Learn more about creating and using encrypted secrets](https://help.github.com/en/actions/automating-your-workflow-with-github-actions/creating-and-using-encrypted-secrets)
    ssh-key: ''

    # Known hosts in addition to the user and global host key database. The public SSH
    # keys for a host may be obtained using the utility `ssh-keyscan`. For example,
    # `ssh-keyscan github.com`. The public key for github.com is always implicitly
    # added.
    ssh-known-hosts: ''

    # Whether to perform strict host key checking. When true, adds the options
    # `StrictHostKeyChecking=yes` and `CheckHostIP=no` to the SSH command line. Use
    # the input `ssh-known-hosts` to configure additional hosts.
    # Default: true
    ssh-strict: ''

    # Whether to configure the token or SSH key with the local git config
    # Default: true
    persist-credentials: ''

    # Relative path under $GITHUB_WORKSPACE to place the repository
    path: ''

    # Whether to execute `git clean -ffdx && git reset --hard HEAD` before fetching
    # Default: true
    clean: ''

    # Number of commits to fetch. 0 indicates all history for all branches and tags.
    # Default: 1
    fetch-depth: ''

    # Whether to download Git-LFS files
    # Default: false
    lfs: ''

    # Whether to checkout submodules: `true` to checkout submodules or `recursive` to
    # recursively checkout submodules.
    #
    # When the `ssh-key` input is not provided, SSH URLs beginning with
    # `git@github.com:` are converted to HTTPS.
    #
    # Default: false
    submodules: ''

    # Add repository path as safe.directory for Git global config by running `git
    # config --global --add safe.directory <path>`
    # Default: true
    set-safe-directory: ''

    # The base URL for the GitHub instance that you are trying to clone from, will use
    # environment defaults to fetch from the same instance that the workflow is
    # running from unless specified. Example URLs are https://github.com or
    # https://my-ghes-server.example.com
    github-server-url: ''
```
<!-- end usage -->

# Scenarios

- [Fetch all history for all tags and branches](#Fetch-all-history-for-all-tags-and-branches)
- [Checkout a different branch](#Checkout-a-different-branch)
- [Checkout HEAD^](#Checkout-HEAD)
- [Checkout multiple repos (side by side)](#Checkout-multiple-repos-side-by-side)
- [Checkout multiple repos (nested)](#Checkout-multiple-repos-nested)
- [Checkout multiple repos (private)](#Checkout-multiple-repos-private)
- [Checkout pull request HEAD commit instead of merge commit](#Checkout-pull-request-HEAD-commit-instead-of-merge-commit)
- [Checkout pull request on closed event](#Checkout-pull-request-on-closed-event)
- [Push a commit using the built-in token](#Push-a-commit-using-the-built-in-token)

## Fetch all history for all tags and branches

```yaml
- uses: actions/checkout@v3
  with:
    fetch-depth: 0
```

## Checkout a different branch

```yaml
- uses: actions/checkout@v3
  with:
    ref: my-branch
```

## Checkout HEAD^

```yaml
- uses: actions/checkout@v3
  with:
    fetch-depth: 2
- run: git checkout HEAD^
```

## Checkout multiple repos (side by side)

```yaml
- name: Checkout
  uses: actions/checkout@v3
  with:
    path: main

- name: Checkout tools repo
  uses: actions/checkout@v3
  with:
    repository: my-org/my-tools
    path: my-tools
```
> - If your secondary repository is private you will need to add the option noted in [Checkout multiple repos (private)](#Checkout-multiple-repos-private)

## Checkout multiple repos (nested)

```yaml
- name: Checkout
  uses: actions/checkout@v3

- name: Checkout tools repo
  uses: actions/checkout@v3
  with:
    repository: my-org/my-tools
    path: my-tools
```
> - If your secondary repository is private you will need to add the option noted in [Checkout multiple repos (private)](#Checkout-multiple-repos-private)

## Checkout multiple repos (private)

```yaml
- name: Checkout
  uses: actions/checkout@v3
  with:
    path: main

- name: Checkout private tools
  uses: actions/checkout@v3
  with:
    repository: my-org/my-private-tools
    token: ${{ secrets.GH_PAT }} # `GH_PAT` is a secret that contains your PAT
    path: my-tools
```

> - `${{ github.token }}` is scoped to the current repository, so if you want to checkout a different repository that is private you will need to provide your own [PAT](https://help.github.com/en/github/authenticating-to-github/creating-a-personal-access-token-for-the-command-line).


## Checkout pull request HEAD commit instead of merge commit

```yaml
- uses: actions/checkout@v3
  with:
    ref: ${{ github.event.pull_request.head.sha }}
```

## Checkout pull request on closed event

```yaml
on:
  pull_request:
    branches: [main]
    types: [opened, synchronize, closed]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
```

## Push a commit using the built-in token

```yaml
on: push
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: |
          date > generated.txt
          git config user.name github-actions
          git config user.email github-actions@github.com
          git add .
          git commit -m "generated"
          git push
```
------------------------------------------------------------------------------------------------------------------

Github Actions Setup-Java --->

# Setup Java

[![Basic validation](https://github.com/actions/setup-java/actions/workflows/basic-validation.yml/badge.svg?branch=main)](https://github.com/actions/setup-java/actions/workflows/basic-validation.yml)
[![Validate Java e2e](https://github.com/actions/setup-java/actions/workflows/e2e-versions.yml/badge.svg?branch=main)](https://github.com/actions/setup-java/actions/workflows/e2e-versions.yml)
[![Validate cache](https://github.com/actions/setup-java/actions/workflows/e2e-cache.yml/badge.svg?branch=main)](https://github.com/actions/setup-java/actions/workflows/e2e-cache.yml)

The `setup-java` action provides the following functionality for GitHub Actions runners:
- Downloading and setting up a requested version of Java. See [Usage](#Usage) for a list of supported distributions.
- Extracting and caching custom version of Java from a local file.
- Configuring runner for publishing using Apache Maven.
- Configuring runner for publishing using Gradle.
- Configuring runner for using GPG private key.
- Registering problem matchers for error output.
- Caching dependencies managed by Apache Maven.
- Caching dependencies managed by Gradle.
- Caching dependencies managed by sbt.
- [Maven Toolchains declaration](https://maven.apache.org/guides/mini/guide-using-toolchains.html) for specified JDK versions.

This action allows you to work with Java and Scala projects.

## V2 vs V1

- V2 supports custom distributions and provides support for Azul Zulu OpenJDK, Eclipse Temurin and AdoptOpenJDK  out of the box. V1 supports only Azul Zulu OpenJDK.
- V2 requires you to specify distribution along with the version. V1 defaults to Azul Zulu OpenJDK, only version input is required. Follow [the migration guide](docs/switching-to-v2.md) to switch from V1 to V2.

## Usage

  - `java-version`: The Java version that is going to be set up. Takes a whole or [semver](#supported-version-syntax) Java version. If not specified, the action will expect `java-version-file` input to be specified.

  - `java-version-file`: The path to the `.java-version` file. See more details in [about `.java-version` file](docs/advanced-usage.md#Java-version-file).
   
  - `distribution`: _(required)_ Java [distribution](#supported-distributions).

  - `java-package`: The packaging variant of the choosen distribution. Possible values: `jdk`, `jre`, `jdk+fx`, `jre+fx`. Default value: `jdk`.

  - `architecture`: The target architecture of the package. Possible values: `x86`, `x64`, `armv7`, `aarch64`, `ppc64le`. Default value: Derived from the runner machine.

  - `jdkFile`: If a use-case requires a custom distribution setup-java uses the compressed JDK from the location pointed by this input and will take care of the installation and caching on the VM.

  - `check-latest`: Setting this option makes the action to check for the latest available version for the version spec.

  - `cache`: Quick [setup caching](#caching-packages-dependencies) for the dependencies managed through one of the predifined package managers. It can be one of "maven", "gradle" or "sbt".

  #### Maven options
  The action has a bunch of inputs to generate maven's [settings.xml](https://maven.apache.org/settings.html) on the fly and pass the values to Apache Maven GPG Plugin as well as Apache Maven Toolchains. See [advanced usage](docs/advanced-usage.md) for more.

  - `overwrite-settings`: By default action overwrites the settings.xml. In order to skip generation of file if it exists set this to `false`.

  - `server-id`: ID of the distributionManagement repository in the pom.xml file. Default is `github`.

  - `server-username`: Environment variable name for the username for authentication to the Apache Maven repository. Default is GITHUB_ACTOR.

  - `server-password`: Environment variable name for password or token for authentication to the Apache Maven repository. Default is GITHUB_TOKEN.

  - `settings-path`: Maven related setting to point to the directory where the settings.xml file will be written. Default is ~/.m2.

  - `gpg-private-key`: GPG private key to import. Default is empty string.

  - `gpg-passphrase`: description: Environment variable name for the GPG private key passphrase. Default is GPG_PASSPHRASE.

  - `mvn-toolchain-id`: Name of Maven Toolchain ID if the default name of `${distribution}_${java-version}` is not wanted.

  - `mvn-toolchain-vendor`: Name of Maven Toolchain Vendor if the default name of `${distribution}` is not wanted.

### Basic Configuration

#### Eclipse Temurin
```yaml
steps:
- uses: actions/checkout@v3
- uses: actions/setup-java@v3
  with:
    distribution: 'temurin' # See 'Supported distributions' for available options
    java-version: '17'
- run: java HelloWorldApp.java
```

#### Azul Zulu OpenJDK
```yaml
steps:
- uses: actions/checkout@v3
- uses: actions/setup-java@v3
  with:
    distribution: 'zulu' # See 'Supported distributions' for available options
    java-version: '17'
- run: java HelloWorldApp.java
```

#### Supported version syntax
The `java-version` input supports an exact version or a version range using [SemVer](https://semver.org/) notation:
- major versions: `8`, `11`, `16`, `17`
- more specific versions: `17.0`, `11.0`, `11.0.4`, `8.0.232`, `8.0.282+8`
- early access (EA) versions: `15-ea`, `15.0.0-ea`, `15.0.0-ea.2`, `15.0.0+2-ea`

#### Supported distributions
Currently, the following distributions are supported:
| Keyword | Distribution | Official site | License
|-|-|-|-|
| `temurin` | Eclipse Temurin | [Link](https://adoptium.net/) | [Link](https://adoptium.net/about.html)
| `zulu` | Azul Zulu OpenJDK | [Link](https://www.azul.com/downloads/zulu-community/?package=jdk) | [Link](https://www.azul.com/products/zulu-and-zulu-enterprise/zulu-terms-of-use/) |
| `adopt` or `adopt-hotspot` | AdoptOpenJDK Hotspot | [Link](https://adoptopenjdk.net/) | [Link](https://adoptopenjdk.net/about.html) |
| `adopt-openj9` | AdoptOpenJDK OpenJ9 | [Link](https://adoptopenjdk.net/) | [Link](https://adoptopenjdk.net/about.html) |
| `liberica` | Liberica JDK | [Link](https://bell-sw.com/) | [Link](https://bell-sw.com/liberica_eula/) |
| `microsoft` | Microsoft Build of OpenJDK | [Link](https://www.microsoft.com/openjdk) | [Link](https://docs.microsoft.com/java/openjdk/faq)
| `corretto` | Amazon Corretto Build of OpenJDK | [Link](https://aws.amazon.com/corretto/) | [Link](https://aws.amazon.com/corretto/faqs/)
| `semeru` | IBM Semeru Runtime Open Edition | [Link](https://developer.ibm.com/languages/java/semeru-runtimes/downloads/) | [Link](https://openjdk.java.net/legal/gplv2+ce.html) |
| `oracle` | Oracle JDK | [Link](https://www.oracle.com/java/technologies/downloads/) | [Link](https://java.com/freeuselicense)

**NOTE:** The different distributors can provide discrepant list of available versions / supported configurations. Please refer to the official documentation to see the list of supported versions.

**NOTE:** AdoptOpenJDK got moved to Eclipse Temurin and won't be updated anymore. It is highly recommended to migrate workflows from `adopt` and `adopt-openj9`, to `temurin` and `semeru` respectively, to keep receiving software and security updates. See more details in the [Good-bye AdoptOpenJDK post](https://blog.adoptopenjdk.net/2021/08/goodbye-adoptopenjdk-hello-adoptium/).

**NOTE:** For Azul Zulu OpenJDK architectures x64 and arm64 are mapped to x86 / arm with proper hw_bitness.

### Caching packages dependencies
The action has a built-in functionality for caching and restoring dependencies. It uses [actions/cache](https://github.com/actions/cache) under hood for caching dependencies but requires less configuration settings. Supported package managers are gradle, maven and sbt. The format of the used cache key is `setup-java-${{ platform }}-${{ packageManager }}-${{ fileHash }}`, where the hash is based on the following files:
- gradle: `**/*.gradle*`, `**/gradle-wrapper.properties`, `buildSrc/**/Versions.kt`, `buildSrc/**/Dependencies.kt`, and `gradle/*.versions.toml`
- maven: `**/pom.xml`
- sbt: all sbt build definition files `**/*.sbt`, `**/project/build.properties`, `**/project/**.scala`, `**/project/**.sbt`

The workflow output `cache-hit` is set to indicate if an exact match was found for the key [as actions/cache does](https://github.com/actions/cache/tree/main#outputs).

The cache input is optional, and caching is turned off by default.

#### Caching gradle dependencies
```yaml
steps:
- uses: actions/checkout@v3
- uses: actions/setup-java@v3
  with:
    distribution: 'temurin'
    java-version: '17'
    cache: 'gradle'
- run: ./gradlew build --no-daemon
```

#### Caching maven dependencies
```yaml
steps:
- uses: actions/checkout@v3
- uses: actions/setup-java@v3
  with:
    distribution: 'temurin'
    java-version: '17'
    cache: 'maven'
- name: Build with Maven
  run: mvn -B package --file pom.xml
```

#### Caching sbt dependencies
```yaml
steps:
- uses: actions/checkout@v3
- uses: actions/setup-java@v3
  with:
    distribution: 'temurin'
    java-version: '17'
    cache: 'sbt'
- name: Build with SBT
  run: sbt package
```

### Check latest

In the basic examples above, the `check-latest` flag defaults to `false`. When set to `false`, the action tries to first resolve a version of Java from the local tool cache on the runner. If unable to find a specific version in the cache, the action will download a version of Java. Use the default or set `check-latest` to `false` if you prefer a faster more consistent setup experience that prioritizes trying to use the cached versions at the expense of newer versions sometimes being available for download.

If `check-latest` is set to `true`, the action first checks if the cached version is the latest one. If the locally cached version is not the most up-to-date, the latest version of Java will be downloaded. Set `check-latest` to `true` if you want the most up-to-date version of Java to always be used. Setting `check-latest` to `true` has performance implications as downloading versions of Java is slower than using cached versions.

For Java distributions that are not cached on Hosted images, `check-latest` always behaves as `true` and downloads Java on-flight. Check out [Hosted Tool Cache](docs/advanced-usage.md#Hosted-Tool-Cache) for more details about pre-cached Java versions.


```yaml
steps:
- uses: actions/checkout@v3
- uses: actions/setup-java@v3
  with:
    distribution: 'temurin'
    java-version: '17'
    check-latest: true
- run: java HelloWorldApp.java
```

### Testing against different Java versions
```yaml
jobs:
  build:
    runs-on: ubuntu-20.04
    strategy:
      matrix:
        java: [ '8', '11', '17' ]
    name: Java ${{ matrix.Java }} sample
    steps:
      - uses: actions/checkout@v3
      - name: Setup java
        uses: actions/setup-java@v3
        with:
          distribution: '<distribution>'
          java-version: ${{ matrix.java }}
      - run: java HelloWorldApp.java
```

### Install multiple JDKs

All versions are added to the PATH. The last version will be used and available globally. Other Java versions can be accessed through env variables with such specification as 'JAVA_HOME_{{ MAJOR_VERSION }}_{{ ARCHITECTURE }}'.

```yaml
    steps:
      - uses: actions/setup-java@v3
        with:
          distribution: '<distribution>'
          java-version: |
            8
            11
            15
```

### Using Maven Toolchains
In the example above multiple JDKs are installed for the same job. The result after the last JDK is installed is a Maven Toolchains declaration containing references to all three JDKs. The values for `id`, `version`, and `vendor` of the individual Toolchain entries are the given input values for `distribution` and `java-version` (`vendor` being the combination of `${distribution}_${java-version}`) by default.

### Advanced Configuration

- [Selecting a Java distribution](docs/advanced-usage.md#Selecting-a-Java-distribution)
  - [Eclipse Temurin](docs/advanced-usage.md#Eclipse-Temurin)
  - [Adopt](docs/advanced-usage.md#Adopt)
  - [Zulu](docs/advanced-usage.md#Zulu)
  - [Liberica](docs/advanced-usage.md#Liberica)
  - [Microsoft](docs/advanced-usage.md#Microsoft)
  - [Amazon Corretto](docs/advanced-usage.md#Amazon-Corretto)
  - [Oracle](docs/advanced-usage.md#Oracle)
- [Installing custom Java package type](docs/advanced-usage.md#Installing-custom-Java-package-type)
- [Installing custom Java architecture](docs/advanced-usage.md#Installing-custom-Java-architecture)
- [Installing custom Java distribution from local file](docs/advanced-usage.md#Installing-Java-from-local-file)
- [Testing against different Java distributions](docs/advanced-usage.md#Testing-against-different-Java-distributions)
- [Testing against different platforms](docs/advanced-usage.md#Testing-against-different-platforms)
- [Publishing using Apache Maven](docs/advanced-usage.md#Publishing-using-Apache-Maven)
- [Publishing using Gradle](docs/advanced-usage.md#Publishing-using-Gradle)
- [Hosted Tool Cache](docs/advanced-usage.md#Hosted-Tool-Cache)
- [Modifying Maven Toolchains](docs/advanced-usage.md#Modifying-Maven-Toolchains)

------------------------------------------------------------------------------------------------------------------