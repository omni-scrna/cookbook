# `one-data` `.h5ad` files are not committed

The dataset `.h5ad` files in this folder are ~1 GB and gitignored
(`samples/one-data/*.h5ad`). The truth-label files and `PROVENANCE.txt`
are committed normally.

## To regenerate the `.h5ad` locally

Run the benchmark in the sibling `split-stages-plan` repo, then from the
cookbook root:

```bash
pixi run samples
```

## Long-term

Once we pick a fixture mechanism (LFS, fetch-on-demand from a release, or
the s3 archival backend), the `.h5ad` can move there.
