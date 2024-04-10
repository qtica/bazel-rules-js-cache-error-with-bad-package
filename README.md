# Description

If a pnpm package provides a tar with directories that are not marked as searchable, then the package is extracted
maintaining the searchless permissions on the directories. Bazel's cache will get permission denied trying to store the
files, and a remote execution service will completely error out at the executor with the same permission denied error.

I believe this is more a bug with the package itself, as it even has an open issue for
it: https://github.com/pngjs/pngjs/issues/381. However, it would be nice if the js rules could gracefully handle this,
considering it appears to not be a problem with npm or pnpm.

# Details

Example of the extracted file structure in bazel-out:

```
bazel-out/k8-fastbuild/bin/node_modules/.aspect_rules_js/pngjs@7.0.0/node_modules/pngjs/coverage:
total 40
drwxr-xr-x 3 <user> <user>  4096 Apr  4 17:47 .
drwxr-xr-x 4 <user> <user>  4096 Apr  4 17:47 ..
drw-r--r-- 2 <user> <user>  4096 Apr 10  2020 lcov-report
-rw-r--r-- 1 <user> <user> 28336 Apr 10  2020 lcov.info

bazel-out/k8-fastbuild/bin/node_modules/.aspect_rules_js/pngjs@7.0.0/node_modules/pngjs/coverage/lcov-report:
total 0
d????????? ? ? ? ?            ? .
d????????? ? ? ? ?            ? ..
-????????? ? ? ? ?            ? base.css
-????????? ? ? ? ?            ? bitmapper.js.html
-????????? ? ? ? ?            ? bitpacker.js.html
-????????? ? ? ? ?            ? block-navigation.js
-????????? ? ? ? ?            ? chunkstream.js.html
-????????? ? ? ? ?            ? constants.js.html
```

We can see that `lcov-report` is not marked as searchable, and this causes the problems seen with bazel cache/remote
execution.

If we look at the contents of the package.tgz itself, we can see that they originally were not marked as searchable:

```shell 
tar -tvf $OUTPUT_BASE/execroot/_main/external/aspect_rules_js\~\~npm\~npm__pngjs__7.0.0/package.tgz
```

```
drw-rw-rw- 0/0               0 2023-02-20 15:58 package
-rw-rw-rw- 0/0            3001 2023-02-20 15:58 package/CHANGELOG.md
-rw-rw-rw- 0/0            1150 2023-02-20 15:58 package/LICENSE
-rw-rw-rw- 0/0            9375 2023-02-20 15:58 package/README.md
-rw-rw-rw- 0/0          572711 2023-02-20 15:59 package/browser.js
drw-rw-rw- 0/0               0 2020-10-24 23:24 package/lib
-rw-rw-rw- 0/0            1812 2023-02-20 15:58 package/package.json
-rw-rw-rw- 0/0            6477 2020-04-16 04:11 package/lib/bitmapper.js
-rw-rw-rw- 0/0            4670 2020-04-16 04:11 package/lib/bitpacker.js
-rw-rw-rw- 0/0            4249 2020-04-16 04:11 package/lib/chunkstream.js
-rw-rw-rw- 0/0             662 2020-04-16 04:11 package/lib/constants.js
-rw-rw-rw- 0/0             853 2020-04-16 04:11 package/lib/crc.js
-rw-rw-rw- 0/0            4279 2020-04-16 04:11 package/lib/filter-pack.js
-rw-rw-rw- 0/0             558 2020-04-16 04:11 package/lib/filter-parse-async.js
-rw-rw-rw- 0/0             483 2020-04-16 04:11 package/lib/filter-parse-sync.js
-rw-rw-rw- 0/0            4798 2020-04-17 03:19 package/lib/filter-parse.js
-rw-rw-rw- 0/0            2366 2020-09-14 03:13 package/lib/format-normaliser.js
-rw-rw-rw- 0/0            2004 2020-04-16 04:11 package/lib/interlace.js
-rw-rw-rw- 0/0            1162 2020-04-16 04:11 package/lib/packer-async.js
-rw-rw-rw- 0/0            1193 2020-04-16 04:11 package/lib/packer-sync.js
-rw-rw-rw- 0/0            3723 2020-04-16 04:11 package/lib/packer.js
-rw-rw-rw- 0/0             372 2020-04-16 04:11 package/lib/paeth-predictor.js
-rw-rw-rw- 0/0            4368 2020-09-14 03:13 package/lib/parser-async.js
-rw-rw-rw- 0/0            2580 2020-09-14 03:13 package/lib/parser-sync.js
-rw-rw-rw- 0/0            7720 2020-04-17 03:19 package/lib/parser.js
-rw-rw-rw- 0/0             252 2020-04-16 04:11 package/lib/png-sync.js
-rw-rw-rw- 0/0            4418 2020-04-16 04:11 package/lib/png.js
-rw-rw-rw- 0/0            3751 2020-04-16 04:11 package/lib/sync-inflate.js
-rw-rw-rw- 0/0            1114 2020-10-24 23:24 package/lib/sync-reader.js
```

## Setup

### Bazel

Bazel Version: `7.1.1`

### rules_js

Version: >`1.4.0`

Error introduced by: https://github.com/aspect-build/rules_js/pull/1538

### pnpm

Dependency on [pngjs](https://github.com/pngjs/pngjs)

### Sample Builds

#### Local Disk Cache

```shell
bazel clean --expunge && bazel build //:.aspect_rules_js/node_modules/pngjs@7.0.0/pkg --disk_cache=./disk_cache
```

Noteable Warning:

```
WARNING: Remote Cache: $OUTPUT_BASE/77389c85a89bd7de7f38028f946643cc/execroot/_main/bazel-out/k8-fastbuild/bin/node_modules/.aspect_rules_js/pngjs@7.0.0/node_modules/pngjs/lib/packer.js (Permission denied)
```

Full Log:

```
INFO: Starting clean (this may take a while). Consider using --async if the clean takes more than several minutes.
Starting local Bazel server and connecting to it...
INFO: Invocation ID: a5626ee6-ab06-45b2-8f3d-82865c8b926e
INFO: Analyzed target //:.aspect_rules_js/node_modules/pngjs@7.0.0/pkg (81 packages loaded, 429 targets configured).
WARNING: Remote Cache: $OUTPUT_BASE/77389c85a89bd7de7f38028f946643cc/execroot/_main/bazel-out/k8-fastbuild/bin/node_modules/.aspect_rules_js/pngjs@7.0.0/node_modules/pngjs/lib/packer.js (Permission denied)
INFO: Found 1 target...
Target //:.aspect_rules_js/node_modules/pngjs@7.0.0/pkg up-to-date:
  bazel-bin/node_modules/.aspect_rules_js/pngjs@7.0.0/node_modules/pngjs
INFO: Elapsed time: 3.807s, Critical Path: 0.07s
INFO: 2 processes: 1 internal, 1 linux-sandbox.
INFO: Build completed successfully, 2 total actions
```

#### Remote Execution

Setup:

1. Create a [platforms/BUILD.bazel](platforms/BUILD.bazel) file
2. Add a platform with the name `remote`
3. Configure a [user.bazelrc](user.bazelrc) with any flags for your RBE service

```shell
bazel build //:.aspect_rules_js/node_modules/pngjs@7.0.0/pkg --config=rbe
```

Noteable Error:

```
ERROR: BUILD.bazel:3:22: Extracting npm package pngjs@7.0.0 failed: (Exit 34): Remote Execution Failure: /mnt/worker/work/0/exec/bazel-out/k8-fastbuild/bin/node_modules/.aspect_rules_js/pngjs@7.0.0/node_modules/pngjs/lib/bitmapper.js (Permission denied)
```

Full Log:

```
INFO: Invocation ID: 015cd95c-bef7-4ac9-b876-9442cb2b0e5a
INFO: Streaming build results to: <REDACTED>
WARNING: Build options --define, --extra_execution_platforms, --host_platform, and 1 more have changed, discarding analysis cache (this can be expensive, see https://bazel.build/advanced/performance/iteration-speed).
INFO: Analyzed target //:.aspect_rules_js/node_modules/pngjs@7.0.0/pkg (1 packages loaded, 430 targets configured).
ERROR: BUILD.bazel:3:22: Extracting npm package pngjs@7.0.0 failed: (Exit 34): Remote Execution Failure: /mnt/worker/work/0/exec/bazel-out/k8-fastbuild/bin/node_modules/.aspect_rules_js/pngjs@7.0.0/node_modules/pngjs/lib/bitmapper.js (Permission denied)
<REDACTED>
Target //:.aspect_rules_js/node_modules/pngjs@7.0.0/pkg failed to build
Use --verbose_failures to see the command lines of failed build steps.
INFO: Elapsed time: 7.007s, Critical Path: 4.74s
INFO: 2 processes: 2 internal.
ERROR: Build did NOT complete successfully
INFO: Streaming build results to: <REDACTED>
```
