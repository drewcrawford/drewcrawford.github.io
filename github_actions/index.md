---
layout: page
title: Github Actions
---
{% raw %}
# Skipping 'rerun' scheduled builds

Suppose you have some expensive CI workflow, long-running tests or the like.  It's too expensive to run on every commit, so you schedule it to run once a day instead:

```yaml
name: daily

on:
  schedule: 
    #8am cst
    - cron: "0 13 * * *"
```

Now you hack on this project all weekend, and then you stop committing for awhile.  **However, the daily build will run whether you committed or not.**  As a consequence, the daily build becomes **more** expensive than the `on: push` version.

Instead, you want some way to run the build each day, *only if there are new commits*.  There are various suggestions for how to do this based on [publishing artifacts](https://github.community/t/skip-github-action-workflow-if-it-has-been-executed-before-on-the-same-commit/16367/3), [parsing dates](https://stackoverflow.com/a/63023602/116834) and other approaches.  However, these seem rather overengineered/finicky to me.

Instead, these 2 steps will cancel a workflow if there's a matching commit sha in the cache:
```yaml
steps:
      - id: cache-sha
        uses: pat-s/always-upload-cache@v2.1.0
        with:
          path: /tmp/nosuchpath
          key: ${{ github.sha }}-${{github.job}}
      - uses: andymckay/cancel-action@0.2
        if: ${{ steps.cache-sha.outputs.cache-hit == 'true' }}
```
GitHub [does purge the cache eventually](https://github.com/actions/cache#cache-limits), so this may sometimes rerun.  However, it works pretty well for me.

Another trick is to do this on a cheap platform like Linux, before spinning up jobs on an expensive platform like macOS:

```yaml
jobs:
  maybe-cancel:
    runs-on: ubuntu-latest
    steps:
      - id: cache-sha
        uses: pat-s/always-upload-cache@v2.1.0
        with:
          path: /tmp/nosuchpath
          key: ${{ github.sha }}-${{github.job}}
      - uses: andymckay/cancel-action@0.2
        if: ${{ steps.cache-sha.outputs.cache-hit == 'true' }}
        
  # make sure our code compiles
  compile-release:
    runs-on: macos-latest
    needs: [maybe-cancel]
    steps:
    - uses: actions/checkout@v2
    - name: run
      run: xcodebuild ...
```

The macOS job will not begin until the linux job confirms a cache miss.  Since linux is 10x cheaper than macOS, this is a significant savings.

# Incremental builds

## xcode

There are several tricks to incremental builds with xcodebuild:

1.  Backup/restore DerivedData with a [actions/cache](https://github.com/actions/cache) action.
2.  Manually tar/untar
    1.  tar with `cfPp [tarfile] --format posix`.  This preserves various attributes used by xcodebuild
    2.  untar with `xvp`.  This preserves various attributes used by xcodebuild
3.  Set mtime on all sourcefiles to a consistent value
4.  Include `-IgnoreFileSystemDeviceInodeChanges=YES` on the xcodebuild CLI

A complete example:

```yaml
steps:
  #first we want to check out our sourcecode; we will also use this as a staging area for caches.
  #note that if we do this too late, our sourcetree will overwrite staging data.
- uses: actions/checkout@v2

#This will get the date in a variety of components that we can concatenate manually.  Goal
#is to find the most recent data, which will speed up our cache time.
- name: Get Date
  id: get-date
  run: |
    echo "::set-output name=minute::$(/bin/date -u "+%M")"
    echo "::set-output name=week::$(/bin/date -u "+%U")"
    echo "::set-output name=date::$(/bin/date -u "+%Y%m%d")"
    echo "::set-output name=hour::$(/bin/date -u "+%H")"
  shell: bash
# Use the data collected above to build up cache keys into buckets in order of preference
- uses: actions/cache@v2
  with:
    path: dd-tar-cache
    key: metaltest-1-${{ steps.get-date.outputs.week }}-${{ steps.get-date.outputs.date }}-${{ steps.get-date.outputs.hour }}-${{ steps.get-date.outputs.minute }}
    restore-keys: |
      metaltest-1-${{ steps.get-date.outputs.week }}-${{ steps.get-date.outputs.date }}-${{ steps.get-date.outputs.hour }}
      metaltest-1-${{ steps.get-date.outputs.week }}-${{ steps.get-date.outputs.date }}
      metaltest-1-${{ steps.get-date.outputs.week }}
      metaltest-1

# Extract our derived data from a tar file, if available.  We need to tar in its own step
# to preserve permissions, use posix notation for higher-resolution timestamps, etc
- name: dd untar
  run: if [ -f dd-tar-cache/dd.tar ]; then tar xvPpf dd-tar-cache/dd.tar; else echo "No cache file"; fi

# Xcode uses the mtime as one of its signals for incremental builds.  When we check out a repository, the mtime
# is initially the time we did the checkout, which means that each build will be 'new'.
# The script below sets the mtime based on the time the file was modified in git.  It's not exact,
# but it's at least *consistent*, which is what we need for incremental builds.
- name: set mtime
  run: |
    set +e
    git submodule foreach 'rev=HEAD; for f in $(git ls-tree -r -t --full-name --name-only "$rev") ; do     touch -t $(git log --pretty=format:%cd --date=format:%Y%m%d%H%M.%S -1 "$rev" -- "$f") "$f"; done'
    rev=HEAD; for f in $(git ls-tree -r -t --full-name --name-only "$rev") ; do     touch -t $(git log --pretty=format:%cd --date=format:%Y%m%d%H%M.%S -1 "$rev" -- "$f") "$f"; done
# Specify a particular version of xcode.  In this example, I'm using xcode 12 beta
- name: xcode-select
  run: sudo xcode-select -s /Applications/Xcode_12_beta.app

# Important to this part, we use a custom `derivedDataPath` and we tell xcode to ignore inode changes.
- name: test
  run: xcodebuild -IgnoreFileSystemDeviceInodeChanges=YES -derivedDataPath localDerivedData -scheme "MyScheme" | xcpretty && exit ${PIPESTATUS[0]}
# tar up the derived data in the appropriate format that we used for untarring above.
- name: dd-tar
  run: mkdir -p dd-tar-cache && tar cfPp dd-tar-cache/dd.tar --format posix localDerivedData
```

## swiftpm

In spite of the fact that actions/cache [lists an example for this usage](https://github.com/actions/cache/blob/main/examples.md#swift---swift-package-manager), I don't believe it is currently possible.

Further reading on this topic:
* https://bugs.swift.org/browse/SR-11760
* https://github.com/actions/cache/pull/159
* https://github.com/apple/swift-llbuild/pull/641#



# Can any of these be placed in a reusable GitHub action?

Not currently, at least not maintainably.  The "cache" action is very complex and cannot be trivially composed with other behavior.  See [#612](https://github.com/actions/runner/pull/612) for adding the "uses" flag to recursively compose actions.

{% endraw %}