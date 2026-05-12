<p align="center">
  <img src="cauldron.png" alt="cookbook" width="240">
</p>

# Contributing to the scRNA-seq benchmark

> **Status:** experimental. We are drafting "good practices" bottom-up. Expect this
> document to change as we learn what flow works.

This repo is a sibling of [`split-stages-plan`](https://github.com/omni-scrna/split-stages-plan)
and is pinned from the org so contributors can find it easily.

It collects three things:

1. **How to write a module** — [`modules-howto.md`](./modules-howto.md)
2. **Sample input files** for every stage — [`samples/`](./samples/)
   so you can run your module in isolation, without re-running the full benchmark.
3. **Conventions** for entrypoints, parameters, outputs, and (eventually) validators.

## Why a separate repo?

- Module authors don't need the full benchmark checkout to get started.
- Sample files are too bulky / too "real-run-specific" to live in the benchmark
  repo proper. Keeping them here lets us regenerate them periodically without
  churning the main repo's history.
- Once we converge on a flow that works, we'll **move** the relevant bits back
  into omnibenchmark core (validators, module scaffolding, sample fetching).

## The contributor flow we're testing

1. Pick a stage you want to contribute a module to (see the [stage map](#stages)).
2. Run `ob create module` with the template that matches your tooling.
3. Pull the matching sample file(s) from `samples/<stage>/`.
4. Run your module locally against the sample. It should produce an output that
   matches the stage's declared `path` pattern.
5. (Optional, for now) run the stage's validator from
   [`split-stages-plan/validators/<stage>/`](https://github.com/omni-scrna/split-stages-plan/tree/main/validators)
   on your output.
6. Open a PR against the benchmark repo bumping the module's `commit:` pin.

## Stages

| Stage | Inputs | Outputs | Sample dir |
|---|---|---|---|
| `one-data` | — | `{dataset}.h5ad`, truth labels | [samples/one-data](./samples/one-data) |
| `two-filter` | `rawdata.h5ad` | `{dataset}_cellids.txt.gz` | [samples/two-filter](./samples/two-filter) |
| `three-normalize` | `rawdata.h5ad`, `filtered.cellids` | `{dataset}_normalized.h5` | [samples/three-normalize](./samples/three-normalize) |
| `four-select` | `normalized.h5` | `{dataset}_normalized_selected.h5` | [samples/four-select](./samples/four-select) |
| `five-pca` | `normalized_selected.h5` | `{dataset}_pcas.tsv` | [samples/five-pca](./samples/five-pca) |
| `graph` | `pcas.tsv` | `{dataset}_neighbors.h5` | [samples/graph](./samples/graph) |
| `cluster` | `neighbors.h5` | `{dataset}_clusters.tsv` | [samples/cluster](./samples/cluster) |

## Open questions

- Where should validators live long-term — here, in the benchmark repo, or in
  each module repo? Today they live in the benchmark repo (`validators/`).
- Should samples be Git LFS, plain commits, or fetched on demand from a release?
  Right now we commit small samples directly. **Hacky, but explicit.**
- Module naming: `<tool>-<role>` (e.g. `scanpy-pca`) vs `<language>-<role>`
  (e.g. `python-pca`). See HOWTO for the current recommendation.
