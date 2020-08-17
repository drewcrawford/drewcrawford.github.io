---
layout: page
title: Github Actions
---

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