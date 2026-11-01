# Spectest's changes to Firecracker

This fork (`specific-dev/firecracker`, branch `spectest`) is the stable
upstream release plus three additions to the snapshot API, used by
[spectest](https://github.com/specific-dev/spectest-internal)'s
per-test fork/snapshot path. Each is opt-in and a request without it
behaves exactly as upstream. The design record, measurements and the
control-plane side live in that repository's `FIRECRACKER.md`.

## `PUT /snapshot/load` — `mem_backend.overlays`

```json
{"mem_backend": {"backend_type": "File", "backend_path": "base.mem",
                 "overlays": ["diff1.mem", "diff2.mem"]}, ...}
```

`File` backend only. After the base file is mapped `MAP_PRIVATE`, each
overlay — a sparse `Diff` memory file of the same size — has every data
extent mapped `MAP_PRIVATE|MAP_FIXED` from the overlay over the same host
address (`vmm/src/vstate/memory.rs::apply_overlay`). The guest sees
base ⊕ overlays, oldest first, later winning; holes fall through. No file
is merged or written, and every microVM restored from the same base maps
the same inode, so their page cache is shared. Dirty tracking is
unaffected: a later `Diff` of the restored VM is one more link in the
chain, which the caller must pass whole. Extents must be page aligned
(filesystem block ≥ page). Rejected with `Uffd`.

## `PUT /snapshot/create` — `fill_gap_pages`

```json
{"snapshot_type": "Diff", "snapshot_path": "...", "mem_file_path": "...",
 "fill_gap_pages": 4}
```

For a `Diff`, a run of at most N clean pages between two dirty runs is
written too, with the guest's current bytes. The file stays a correct
overlay; it has fewer, larger data extents (a restore maps one region per
extent) at the cost of extra bytes. The slot dump is two passes: collect
dirty runs merging across gaps up to the limit, then write each run.
Default 0 (upstream behaviour).

## `PUT /snapshot/create` — `sync_snapshot_files`

Same name and meaning as upstream's post-1.16 option: `false` skips the
`fsync` of the state and memory files. Default `true`.

## Maintenance

- Point releases: merge upstream's `firecracker-v1.x` branch into
  `spectest`. A new minor: rebase `spectest` onto the new tag.
- The additions touch `vmm/src/vmm_config/snapshot.rs`,
  `vmm/src/persist.rs`, `vmm/src/vstate/{memory,vm}.rs` and the API
  request tests. Unit tests: `cargo test -p vmm dump_dirty` (includes a
  gap-fill test and the dump_dirty proptest oracle).
- Spectest builds this for the host's glibc target
  (`cargo build --release --target <arch>-unknown-linux-gnu -p
  firecracker`; `libseccomp` dev package required) and pins the commit as
  `FC_REF` in its `scripts/local-vms-setup.sh`.

## Snapshot mapping read-ahead

Base snapshot mappings and every sparse overlay extent use `MADV_RANDOM`.
A later `MAP_FIXED` overlay creates a new VMA, so advising only the base is
insufficient. This hint leaves file bytes, private copy-on-write, dirty tracking,
and snapshot serialization unchanged. It does not change block-device policy or
advise anonymous guest RAM. Advice failure is logged without failing restoration.

In a live Harmony run on 2026-11-01, vCPU threads accounted for 87–98% of sampled
Firecracker physical reads (up to 4.85 GB/s), with zero read syscalls and thousands
of major faults. The VM cgroup was at `memory.high`, repeatedly reclaiming file
pages. A separate cold-file probe touching 1 MiB of scattered pages read 512 MiB
with normal advice and 1 MiB with random advice. This establishes a read-amplification
mechanism, not a measured production improvement. Production performance still
requires a controlled workload comparison after the compatible runtime rollout.

`test_snapshot_random_access_preserves_overlays_and_cow` verifies the real VMA
hint, sparse holes, later-overlay precedence and private writes without KVM.
