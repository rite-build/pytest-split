# Fork Changes (rite-build/pytest-split)

Forked from [jerry-git/pytest-split](https://github.com/jerry-git/pytest-split) at version **0.11.0** (commit `19abca7`).

## Interleaved Duration Sorting (`feat/sort-groups-by-duration`)

### Problem

When using `pytest-xdist` with multiple workers per shard, the order of tests within each shard affects wall-clock time. Without reordering, the longest tests can be queued behind short tests or batched together on a single worker, creating a bottleneck.

Specifically, `pytest-xdist --dist load` dispatches an initial batch of consecutive tests to each worker (minimum chunk size of 2, hardcoded in xdist). If the two longest tests are adjacent in the collection, they land on the same worker.

### Changes

**New CLI flags** in `plugin.py`:

| Flag | Default | Description |
|---|---|---|
| `--sort-groups-by-duration` | Enabled | Reorder tests within each shard by duration |
| `--no-sort-groups-by-duration` | — | Preserve original collection order |

**Sorting logic** in `pytest_collection_modifyitems` (after the split algorithm assigns tests to the shard):

1. Sort tests by cached duration descending (longest first)
2. Interleave using a zigzag pattern: `[longest, shortest, 2nd longest, 2nd shortest, ...]`

```
Collection order:     [A, B, C, D, E, F, G, H]  (arbitrary)
After desc sort:      [100s, 80s, 60s, 40s, 20s, 5s, 2s, 1s]
After interleaving:   [100s, 1s, 80s, 2s, 60s, 5s, 40s, 20s]
```

This ensures xdist's initial chunk-of-2 dispatch pairs each long test with a short one:

- Worker 0 gets: `[100s, 1s]` → 101s
- Worker 1 gets: `[80s, 2s]` → 82s
- Worker 2 gets: `[60s, 5s]` → 65s
- Worker 3 gets: `[40s, 20s]` → 60s

Tests with unknown durations are assigned the average duration of all known tests.

### Files Modified

- `src/pytest_split/plugin.py` — added CLI options and sorting/interleaving logic
- `tests/test_plugin.py` — added `TestSortGroupsByDuration` test class; updated existing tests to use `--no-sort-groups-by-duration` where order-sensitive

### Recommended xdist Flags

For best results, combine with:

```
-n auto --dist load --maxschedchunk 1
```

- `--dist load`: central work queue that respects collection order
- `--maxschedchunk 1`: sets initial dispatch chunk to 2 (xdist minimum), maximizing the benefit of interleaving

### Upstream Compatibility

No upstream APIs or behaviors are changed. The sorting is additive and can be disabled with `--no-sort-groups-by-duration` to restore the original behavior. All upstream tests pass with the flag disabled.
