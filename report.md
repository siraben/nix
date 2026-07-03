# Nix perf thread validation report

Date: 2026-07-03

Checkout:

- Repository: `/home/siraben/nix2`
- Branch: `siraben/perf2`
- Commit: `b41781ed0` (`upstream/master` at measurement time)
- Remotes: `origin = git@github.com:siraben/nix.git`, `upstream = git@github.com:NixOS/nix.git`
- Tested Nix: `/nix/store/19awgimp3f1m9dh096xxmg7kx675cypq-nix-2.35.0pre20260702_b41781e/bin/nix`
- Host: AMD Ryzen 9 5950X, 32 logical CPUs, btrfs for `/home` and `/nix/store`
- Kernel perf setting: `/proc/sys/kernel/perf_event_paranoid = 2`; user-space perf samples worked

The original NixOS configuration path from the chat, `../../projects/nixos/nixos.nix`, was not present from this checkout, so the eval workload below uses this repository's flake package attribute:

```sh
nix eval --raw .#packages.x86_64-linux.nix.name
```

All timing commands were pinned with `taskset -c 2`, used warmups, and were run through `nix shell` where extra tools were needed. Raw artifacts are under `/tmp/nix-perf2`.

## Summary

| Claim | Verdict |
| --- | --- |
| `perf record -g nix eval ...` is a sane starting point | Mostly valid, with caveats |
| Pure and impure eval can perform differently | Source-valid; not significant for the comparable flake workload measured here |
| Profile current master rather than old releases | Valid; measurements used `upstream/master` |
| `packaging/components.nix` has easy packaging performance wins | Valid as an in-tree optimization point; not A/B measured here |
| Pure eval copies/hashes sources; SourceAccessor layering matters | Valid |
| Tarball cache performance can worsen over time; MIDX maintenance helps | Validated on a copied cache |
| XFS-on-ZFS plus SQLite VACUUM can help | VACUUM validated; XFS-on-ZFS not reproducible on this btrfs host |
| Bigger fish: writeback store, reflinks, builder fork, event loop blocking | Source-valid as architectural targets; not all are covered by eval profiling |
| 2.35 has several relevant fixes | Partly valid; this checkout includes 2.34 perf notes and 2.35-pre source changes |

## Profiling approach

The thread's basic approach is reasonable:

```sh
perf record -g result/bin/nix eval ...
perf script | inferno-collapse-perf > nix.profile
```

But for accurate results:

- Build and profile the exact Nix revision under investigation, not `nixpkgs#nixVersions.latest` unless that is intentionally the target.
- Pin the command to a CPU for timing stability.
- Separate pure and impure eval measurements. They exercise different filesystem access paths.
- Warm caches deliberately, or measure cold-cache behavior deliberately. Do not mix them.
- Prefer `--call-graph dwarf,...` or a frame-pointer/debug build when flamegraph stack quality matters. The optimized LTO build produced useful top frames, but the default collapsed stack had many `[unknown]` frames.

For the measured flake eval, `perf report` showed the time dominated by evaluator work such as `ExprVar::eval`, `ExprCall::eval`, `prim_derivationStrict`, string coercion, allocation, and source path/fingerprint paths under `copyPathToStore`. That matches an eval-heavy workload rather than a build or substitution workload.

Artifacts:

- `/tmp/nix-perf2/nix-eval-flake.data`
- `/tmp/nix-perf2/nix-eval-flake-dwarf.data`
- `/tmp/nix-perf2/nix.profile`
- `/tmp/nix-perf2/nix-dwarf.profile`

## Pure vs impure eval

Comparable flake eval timing:

| Mode | Mean | Stddev | Range |
| --- | ---: | ---: | ---: |
| pure | 3.561 s | 0.136 s | 3.420-3.885 s |
| impure | 3.499 s | 0.268 s | 3.161-4.203 s |

Hyperfine summary: impure was `1.02 +/- 0.09` times faster than pure, which is not a meaningful difference for this workload.

However, the source confirms that pure and impure eval are not equivalent. In `src/libexpr/eval.cc`, pure mode uses the mounted store filesystem, while impure mode wraps the real filesystem and store filesystem in a union accessor:

- `storeFS = makeMountedSourceAccessor(...)`
- pure: `storeFS.cast<SourceAccessor>()`
- impure: `makeUnionSourceAccessor({getFSSourceAccessor(), storeFS})`
- then both are wrapped with `makeCachingSourceAccessor(...)`
- pure/restricted eval also adds `AllowListSourceAccessor`

Direct `--expr` access to `./.` failed in pure mode on this machine with the expected error:

```text
access to absolute path '/home/siraben/nix2' is forbidden in pure evaluation mode (use '--impure' to override)
```

So the chat warning is correct: pure/impure eval can have materially different behavior. The comparable flake workload just did not show a large timing delta here.

## Packaging flags

The referenced block in `packaging/components.nix:184-194` is a real packaging optimization:

- It documents GCC's weaker interprocedural optimization behavior for PIC/shared-library code.
- It adds `-fno-semantic-interposition` and `-Wl,-Bsymbolic-functions` for GNU toolchains.
- It notes Clang already inlines differently here.

I did not run a clean A/B rebuild with these flags removed, so this is source-validated rather than empirically quantified in this report.

## SourceAccessor complexity

The SourceAccessor point is valid. The searched tree contains 36 `SourceAccessor` classes/structs and 55 construction sites or factories matching `make*SourceAccessor`/`SourceAccessor::create`.

Relevant source evidence:

- `src/libexpr/eval.cc:247-292` builds different pure/impure root filesystems and layers cache/access control on top.
- `src/libutil/mounted-source-accessor.cc` implements mounted accessors.
- `src/libutil/union-source-accessor.cc` implements union accessors.
- `src/libutil/caching-source-accessor.cc` implements caching accessors.
- `src/libfetchers/filtering-source-accessor.cc` implements allow-list/filtering accessors.
- `src/libfetchers/git-utils.cc` adds Git-specific source accessors and export-ignore filtering.

The perf profile also showed source path/fingerprint and `copyPathToStore` frames, so source access is visible in this eval workload.

## Tarball cache maintenance

The real cache was not modified. I copied it to `/tmp/nix-perf2/tarball-cache-v2` and ran the maintenance sequence on the copy.

Initial copied cache:

- Size: 322 MiB
- Packed objects: 275,097
- Packs: 155
- Multi-pack-index file: absent

After:

```sh
git -C /tmp/nix-perf2/tarball-cache-v2 multi-pack-index write
git -C /tmp/nix-perf2/tarball-cache-v2 multi-pack-index repack
git -C /tmp/nix-perf2/tarball-cache-v2 multi-pack-index expire
```

Result:

- Packed objects: 274,981
- Packs: 1
- Pack size: 183.52 MiB
- Multi-pack-index: present

Lookup benchmark, using 10,000 randomly selected object IDs:

| State | Mean | Stddev | Range |
| --- | ---: | ---: | ---: |
| before maintenance | 97.3 ms | 10.8 ms | 86.2-136.7 ms |
| after maintenance | 59.7 ms | 7.9 ms | 50.9-82.1 ms |

Verdict: valid. On this cache, the suggested maintenance improved the object lookup benchmark by about 1.6x and reduced disk usage substantially.

This also matches `doc/manual/source/release-notes/rl-2.34.md`, which documents that `tarball-cache-v2` currently lacks automatic maintenance and suggests the same `git multi-pack-index` sequence.

## SQLite VACUUM and filesystem claim

This host is btrfs, not XFS on ZFS, so I could not reproduce the XFS-on-ZFS part of the claim.

The SQLite VACUUM part did validate strongly on a copied Nix database.

Live DB state, read only:

- `/nix/var/nix/db/db.sqlite`: 3.4 GiB
- page count: 883,565
- freelist count: 765,498
- page size: 4096
- WAL mode enabled

Copied DB before VACUUM:

- Size: 3.4 GiB
- page count: 883,565
- freelist count: 765,494

Copied DB after `VACUUM`:

- Size: 347 MiB
- page count: 88,665
- freelist count: 0

Simple query benchmark on the copy:

```sql
select count(*) from ValidPaths;
select count(*) from Refs;
```

| State | Mean | Stddev | Range |
| --- | ---: | ---: | ---: |
| before VACUUM | 84.7 ms | 42.8 ms | 64.1-230.5 ms |
| after VACUUM | 20.7 ms | 2.9 ms | 18.1-28.4 ms |

The pre-VACUUM query timing was noisy, but the file-size and freelist results are unambiguous. VACUUM would likely help this machine's store DB too, but I did not modify `/nix/var/nix/db/db.sqlite`.

There is also current ZFS-specific SQLite code in `src/libstore/sqlite.cc:68-83`, which works around a ZFS issue where truncating `db.sqlite-shm` can randomly take seconds. That supports the broader filesystem sensitivity claim, even though this host cannot test the specific XFS-on-ZFS setup.

## Larger architectural targets

These are valid targets, but most are outside what an eval-only perf profile can measure.

Reflinking:

- `src/libstore/unix/build/derivation-builder.cc:1412-1419` serializes/deserializes outputs to create a fresh copy.
- The comment explicitly says: `TODO: Use copyRecursive here and make use of reflinking.`
- `src/libutil/file-system.cc:588-620` uses `std::filesystem::copy`/`copy_file`, with no explicit reflink path in that helper.

Builder fork:

- `src/libutil/unix/processes.cc:214-219` calls `vfork()` or `fork()`.
- `src/libutil/unix/processes.cc:236-292` routes `startProcess` through clone or `doFork`.
- `src/libstore/unix/build/derivation-builder.cc:655-658` starts the builder through `startProcess`.
- `src/libstore/linux/build/linux-derivation-builder.cc:574-581` starts the sandboxed child through `startProcess` with clone flags.

Worker event loop blocking:

- `src/libstore/build/worker.cc:320-335` calls `goal->work()` synchronously inside the worker event loop.
- The surrounding trace message reports how long each goal held the event loop.
- I did not catch a seconds-long hold in this eval workload, but the code shape supports the claim as an architectural risk for build/substitution workloads.

Writeback store:

- This checkout has `local-overlay-store` support and documentation, but I did not find an implementation literally named `writeback store`.
- Treat the thread point as a design direction, not a measured conclusion from this report.

## What I would change in future profiling

For eval profiling:

```sh
nix build .#nix -L --no-link --print-out-paths
taskset -c 2 perf record -F 99 --call-graph dwarf,16384 -- \
  /path/to/built-nix/bin/nix eval --raw .#packages.x86_64-linux.nix.name
perf report --stdio -i perf.data
perf script -i perf.data | inferno-collapse-perf > nix.profile
```

For stable timing:

```sh
hyperfine --warmup 3 --runs 12 \
  'taskset -c 2 /path/to/nix eval --raw ...'
```

For pure/impure comparisons, use semantically comparable commands. Direct filesystem path expressions and flake eval do not exercise the same access path.

## Bottom line

The original approach is sane for exploration, but accurate conclusions require pinning the Nix revision, pinning CPU placement, separating pure and impure eval, controlling cache state, and using a call graph mode that unwinds the optimized binary well enough.

The strongest measured validations here were tarball-cache maintenance and SQLite VACUUM. The pure/impure warning is source-valid, but the measured flake workload did not show a significant difference. The architectural points around SourceAccessor layering, reflinks, builder process creation, and event-loop blocking are supported by the current source, but they need targeted build/substitution workloads rather than this eval profile to quantify.
