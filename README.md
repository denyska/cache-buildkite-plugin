# Cache Buildkite Plugin [![Version badge](https://img.shields.io/badge/cache-v2.3.9-blue?style=flat-square)](https://buildkite.com/plugins) [![Build status](https://badge.buildkite.com/eb76936a02fe8d522fe8cc986c034a6a8d83c7ec75e607f7bb.svg)](https://buildkite.com/gencer/buildkite-cache)


### Tarball, Rsync & S3 Cache Kit for Buildkite. Supports Linux, macOS and Windows*

&ast; Windows requires **Git for Windows 2.25 or later**.


A [Buildkite plugin](https://buildkite.com/docs/agent/v3/plugins) to restore and save
directories by cache keys. For example, use the checksum of a `.resolved` or `.lock` file
to restore/save built dependencies between independent builds, not just jobs.

With tarball or rsync, if source folder has changes, this will not fail your build, instead will surpress, notify and continue.

For S3, Instead of sync thousands of files, It just creates a tarball before S3 operation then copy this tarball to s3 at one time. This will reduce both time and cost on AWS billing.

Plus, In addition to tarball & rsync, we also do not re-create another tarball for same cache key if it's already exists.

# Backends

Please see `lib/backends/*.sh` for available backends. You can fork, add your backend then send a PR here.

Available backends and their requirements:

| **Backend** | **Linux (GNU)**                             | **macOS (BSD)**                             | **Windows**    |
| ----------- | ------------------------------------------- | ------------------------------------------- | -------------- |
| `tarball`   | tar<br />sha1sum                            | tar<br />shasum                             | Same as Linux  |
| `rsync`     | rsync<br />sha1sum                          | rsync <br />shasum                          | Same as Linux* |
| `s3`        | aws-cli (`>= 1, ~> 2`)<br />tar<br/>sha1sum | aws-cli (`>= 1, ~> 2`)<br />tar<br />shasum | Same as Linux  |

### Windows Support

If you install **Git for Windows 2.25 or later**, you will benefit all features of Cache on Windows. Make sure you've added `bash.exe` into your `PATH`.

&ast; Rsync on Windows requires https://itefix.net/cwrsync. To be clear, except `rsync`, you can use `s3` and `tarball` on Windows without an additional app.

# S3

S3 backend uses **AWS CLI** v**1** or v**2** to copy and download from/to S3 bucket or s3-compatible bucket. To be precisely, backend simply uses `aws s3 cp` command for all operations. Before that, we do `head-object` to check existence of the cache key on target bucket. While tarball is the default backend, S3 backend is heavily tested and ready for use in production. See some examples below for S3 backend.

```yml
steps:
  - plugins:
    - gencer/cache#v2.3.9:
        backend: s3
        key: "v1-cache-{{ runner.os }}-{{ checksum 'Podfile.lock' }}"
        s3:
          profile: "other-profile" # Optional. Defaults to `default`.
          bucket: "s3-bucket"
          compress: true # Create tar.gz instead of .tar (Compressed) Defaults to `false`.
          class: STANDARD # Optional. Defaults to empty which is usually STANDARD or based on policy.
          args: '--option 1' # Optional. Defaults to empty. Any optional argument that can be passed to aws s3 cp command.
        paths:
          - 'Pods/'
          - 'Rome/'
```

The paths are synced using Amazon S3 into your bucket using a structure of
`organization-slug/pipeline-slug/cache_key.tar`, as determined by the Buildkite environment
variables.

### S3-compatible Providers

Use `endpoint` and `region` fields to pass host and region parameters to be able to use S3-compatible providers. For example:

```yml
steps:
  - plugins:
    - gencer/cache#v2.3.9:
        backend: s3
        key: "v1-cache-{{ runner.os }}-{{ checksum 'Podfile.lock' }}"
        s3:
          bucket: "s3-bucket"
          endpoint: "https://s3.nl-ams.scw.cloud"
          region: "nl-ams" # Optional. Defaults to `host` one.
          # Alternative: If you strictly need to specify host and region manually, then use like this:
          # args: "--endpoint-url=https://s3.nl-ams.scw.cloud --region=nl-ams"
        paths:
          - 'Pods/'
          - 'Rome/'
```

Or, alternatively you can define them in your `environment` file like this:

```bash
export BUILDKITE_PLUGIN_CACHE_S3_ENDPOINT="https://s3.nl-ams.scw.cloud"
# Optionally specify region:
export BUILDKITE_PLUGIN_CACHE_S3_REGION="nl-ams"
```

### Storage Class

You can pass `class` option for the following classes:

- `STANDARD`
- `STANDARD_IA`
- `ONEZONE_IA`
- `INTELLIGENT_TIERING`

Default value will always be empty. This means, AWS or Compatible provider will use its default value for stored object or a value that given by Lifecycle policy.

### Additional Arguments

You can pass `args` argument with required options. This arguments will be added to the end of `s3 cp` command. Therefore please do not add following options:

- `--storage-class`
- `--profile`
- `--endpoint-url`
- `--region`

However, If you do not specify `profile`, `endpoint`, `region` and `class` via YAML configuration, then you can pass those arguments to the `args`.

# rsync

You can also use rsync to store your files using the `rsync` backend. Files will neither compressed nor packed.

```yml
steps:
  - plugins:
    - gencer/cache#v2.3.9:
        backend: rsync
        key: "v1-cache-{{ runner.os }}-{{ checksum 'Podfile.lock' }}"
        rsync:
          path: '/tmp/buildkite-cache' # Defaults to /tmp with v2.3.9+
        paths:
          - 'Pods/'
          - 'Rome/'
```

The paths are synced using `rsync_path/cache_key/path`. This is useful for maintaining a local
cache directory, even though this cache is not shared between servers, it can be reused by different
agents/builds.

# tarball

You can also use tarballs to store your files using the `tarball` backend. Files will not be compressed but surely packed into single archive.

**This is the Default and Recommended backend for cache**

```yml
steps:
  - plugins:
    - gencer/cache#v2.3.9:
        backend: tarball # Optional. Default `backend` is already set to `tarball` 
        key: "v1-cache-{{ runner.os }}-{{ checksum 'Podfile.lock' }}"
        tarball:
          path: '/tmp/buildkite-cache' # Defaults to /tmp with v2.3.9+
          max: 7 # Optional. Removes tarballs older than 7 days.
          compress: true # Create tar.gz instead of .tar (Compressed) Defaults to `false`.
        paths:
          - 'Pods/'
          - 'Rome/'
```

The paths are synced using `tarball_path/cache_key.tar`. This is useful for maintaining a local
cache directory, even though this cache is not shared between servers, it can be reused by different
agents/builds.

### Cache Key Templates

The cache key is a string, which support a crude template system. Currently `checksum` is
the only command supported for now. It can be used as in the example above. In this case
the cache key will be determined by executing a _checksum_ (actually `sha1sum`) on the
`Gemfile.lock` file, prepended with `v1-cache-{{ runner.os }}-`.

### Hashing (checksum) against directory

Along with lock files, you can calculate directory that contains multiple files or recursive directories and files.

```yml
steps:
  - plugins:
    - gencer/cache#v2.3.9:
        backend: tarball # Optional. Default `backend` is already set to `tarball` 
        key: "v1-cache-{{ runner.os }}-{{ checksum './app/javascript' }}" # Calculate whole 'app/javascript' recursively
        tarball:
          path: '/tmp/buildkite-cache' # Defaults to /tmp with v2.3.9+
          max: 7 # Optional. Removes tarballs older than 7 days. 
          compress: true # Create tar.gz instead of .tar (Compressed) Defaults to `false`.
        paths:
          - 'Pods/'
          - 'Rome/'
```

For example, you can calculate total checksum of your javascript folder to skip build, If the source didn't changed.

Note: Before hashing files, we do "sort". This provides exact same sorted and hashed content against very same directory between builds.

### Skip Cache on PRs

You can skip caching on Pull Requests (Merge Requests) by simply adding `pr: false` to the cache plugin. For example;

```yml
steps:
  - plugins:
    - gencer/cache#v2.3.9:
        backend: s3
        key: "v1-cache-{{ runner.os }}-{{ checksum 'Podfile.lock' }}"
        pr: false # Default to `true` which is do cache on PRs.
        s3:
          profile: "other-profile" # Optional. Defaults to `default`.
          bucket: "s3-bucket"
          compress: true # Create tar.gz instead of .tar (Compressed) Defaults to `false`.
          class: STANDARD # Optional. Defaults to empty which is usually STANDARD or based on policy.
          args: '--option 1' # Optional. Defaults to empty. Any optional argument that can be passed to aws s3 cp command.
        paths:
          - 'Pods/'
          - 'Rome/'
```

Or you can set this by Environment:

```
#!/bin/bash

export BUILDKITE_PLUGIN_CACHE_PR=false
```

### Multiple usages in same pipeline

```yaml
cache: &cache
  key: "v1-cache-{{ runner.os }}-{{ checksum 'yarn.lock' }}"
  backend: s3
  pr: false
  s3:
    bucket: cache-bucket
  paths:
    - node_modules
    # If you have sub-dir then use:
    # - **/node_modules

steps:
  - name: ':jest: Run tests'
    key: jest
    command: yarn test --runInBand
    plugins:
      - gencer/cache#v2.3.9: *cache
  - name: ':istanbul: Run Istanbul'
    key: istanbul
    depends_on: jest
    command: .buildkite/steps/istanbul.sh
    plugins:
      - gencer/cache#v2.3.9: *cache
```

### Usage with docker

Put cache plugin **before** `docker` or `docker-compose` plugins. Let's cache do the rest restoring and caching afterwards.

```yaml
steps:
  - name: ':jest: Run tests'
    key: jest
    command: yarn test --runInBand
    plugins:
      - gencer/cache#v2.3.9: # Define cache *before* docker plugins.
        backend: s3
        key: "v1-cache-{{ runner.os }}-{{ checksum 'Podfile.lock' }}"
        pr: false
        s3:
          bucket: s3-bucket
        paths:
          - Pods/
      - docker#v3.7.0: ~ # Use your config here
      - docker-compose#3.7.0: ~ # Or compose. Use your config here
```

### Auto deletion old caches

To keep caches and delete them in _for example_ 7 days, use tarball backend and use `max`. On S3 side, please use S3 Policy for this routine. Each uploaded file to S3 will be deleted according to your file deletion policy.

**For S3**, Due to expiration policy, we just re-upload the same tarball to refresh expiration date. As long as you use the same cache, S3 will not delete it. Otherwise, It will be deleted from S3-side not used in a manner time.

### Globs on paths

You can use glob pattern in paths (to be cached) after `v2.1.x`

## Roadmap

+ Google Cloud Cache Support.

Original work by [@danthorpe](https://github.com/danthorpe/cache-buildkite-plugin)

Copyright (C) 2021 Gencer W. Genç.

Licensed as **MIT**.
