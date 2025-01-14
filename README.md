# docker-volume-cache-action

A GitHub Action to cache and restore Docker Volumes

## Why?

It is notoriously tricky to cache Docker volume data, especially with GitHub's
native [actions/cache](https://github.com/actions/cache/) action. This combined
action provides a `BYK/docker-volume-cache-action/save` and `BYK/docker-volume-cache-action/restore`
respectively for making this easy and painless.

If you have the time, you can read https://byk.im/posts/docker-volume-caching-gha/

## Usage

```yaml
name: Test
on:
  push:
    branches:
      - "main"
  pull_request:

defaults:
  run:
    shell: bash
jobs:
  test:
    steps:
      - name: Checkout latest release
        uses: actions/checkout@v4

      - name: Compute Docker Volume Cache Key
        id: cache_key
        run: |
          MY_IMAGE_MD5=$(docker run --rm --entrypoint bash MY_DOCKER_IMAGE -c 'ls -Rv1rpq some/directory' | md5sum | cut -d ' ' -f 1)
          echo "MY_IMAGE_MD5=$MY_IMAGE_MD5" >> $GITHUB_OUTPUT

      - name: Restore DB Volumes Cache
        id: restore_cache
        uses: BYK/docker-volume-cache-action/restore@main
        with:
          key: docker-volumes-v1-${{ steps.cache_key.outputs.MY_IMAGE_MD5 }}
          restore-keys: |
            docker-volumes-v1-
          volumes: |
            volume-name-1
            volume-name-2

      - name: Long test using volumes
        if: steps.restore_cache.outputs.cache-hit != 'true'
        run: |
          ./run_tests.sh

      - name: Save DB Volumes Cache
        if: steps.restore_cache.outputs.cache-hit != 'true'
        uses: BYK/docker-volume-cache-action/save@main
        with:
          key: ${{ steps.restore_cache.outputs.cache-primary-key }}
          volumes: |
            volume-name-1
            volume-name-2
```

## Documentation

### Restore action

BYK/docker-volume-cache-action/restore

#### Inputs

* `key` - An explicit key for a cache entry.
* `volumes` - A list of Docker volume names to restore
* `restore-keys` - An ordered list of prefix-matched keys to use for restoring stale cache if no cache hit occurred for key.
* `fail-on-cache-miss` - Fail the workflow if cache entry is not found. Default: `false`

### Outputs

* `cache-hit` - A boolean value to indicate an exact match was found for the key.
* `cache-primary-key` - Cache primary key passed in the input to use in subsequent steps of the workflow.
* `cache-matched-key` - Key of the cache that was restored, it could either be the primary key on cache-hit or a partial/complete match of one of the restore keys.

> **Note**
`cache-hit` will be set to `true` only when cache hit occurs for the exact `key` match. For a partial key match via `restore-keys` or a cache miss, it will be set to `false`.

### Save action

BYK/docker-volume-cache-action/save

#### Inputs

* `key` - An explicit key for a cache entry.
* `volumes` - A list of Docker volume names to cache
* `upload-chunk-size` - The chunk size used to split up large files during upload, in bytes

#### Outputs

This action has no outputs.
