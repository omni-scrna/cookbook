# HOWTO: create a module

A **module** is a small Git repo that exposes one **entrypoint** — a script
that implements one stage of the benchmark. The benchmark pins your module
in `benchmark_conda.yaml` via `repository.url`, `repository.commit`, and
`repository.entrypoint`.

Start with **one entrypoint per module**. Bundling several into one repo is
useful only when they genuinely share code (see [Module shapes](#module-shapes)).

## 1. Scaffold

```bash
pip install omnibenchmark
ob create module --stage <stage-id>
```

## 2. Basic layout

```
my-pca/
├── omnibenchmark.yaml      # entrypoints: { pca: pca.py }
├── pca.py                  # argparse → src.run_pca → write
├── src/                    # functions imported by pca.py and tests/
├── tests/                  # pytest, imports from src/
├── envs/my-pca.yml         # auto-exported from pixi (consumed by the benchmark)
├── pixi.toml               # deps + tasks
└── README.md
```

**Rule:** keep `pca.py` thin — argparse and glue. Real logic goes in `src/`
so `tests/` can import it directly.

```python
# pca.py
import sys; from pathlib import Path
sys.path.insert(0, str(Path(__file__).parent / "src"))
from cli import build_pca_parser     # noqa: E402
from schemas import Embedding        # noqa: E402

def main() -> None:
    args = build_pca_parser().parse_args()
    emb = run_pca(args)              # defined in src/
    emb.write(Path(args.output_dir) / f"{args.name}_pcas.tsv")

if __name__ == "__main__":
    main()
```

## 3. Module metadata: per repo `omnibenchmark.yaml`

Maps an entrypoint **name** to a shell command:

```yaml
entrypoints:
  pca: pca.py
```

The benchmark side picks it with `repository.entrypoint: pca`.

If your module has a single usage, use `entrypoints: default: pca.py`.

Entrypoints are preferred over dispatching on module arguments.

## 4. CLI contract

The benchmark invokes your script with:

- `--<input-id> <path>` per stage input,
- `--<param-name> <value>` per parameter,
- `--output_dir <dir>` and `--name <id>`.

Write outputs into `--output_dir`, named exactly as the stage's `path`
template declares (e.g. `{name}_pcas.tsv`). Flag names with dots like
`--pcas.tsv` are valid — use `dest=` in argparse.

## 5. Test against a sample

The cookbook ships one real input per stage. You can [use those samples](https://github.com/btraven00/cookbook/blob/main/samples/INDEX.md) to quickly test your modules if you dont' want to run the full benchmark:

```bash
git clone https://github.com/omni-scrna/cookbook.git ../cookbook

python pca.py \
  --normalized_selected.h5 ../cookbook/samples/four-select/datasets_normalized_selected.h5 \
  --solver arpack --n_components 50 --random_seed 42 \
  --output_dir /tmp/myrun --name datasets
```

If `/tmp/myrun/datasets_pcas.tsv` appears, you're done (well, assuming your module works as intended). See
[`samples/INDEX.md`](./samples/INDEX.md) for what each sample is.

## 6. Open a PR

Bump `commit:` (or pin a branch/tag) in `benchmark_conda.yaml` and set
`entrypoint:` to your declared name if your module uses entrypoints.

---

## Pixi workflow

`pixi.toml` is the source of truth for deps. The benchmark consumes
`envs/<module>.yml` instead — keep it auto-generated:

```toml
[tasks]
test       = "pytest"
lint       = "ruff check ."
typecheck  = "ty check"
export-env = "pixi workspace export conda-environment ... envs/my-pca.yml"
```

Run `pixi run export-env` whenever deps change. Don't hand-write the conda yaml.

## Module shapes

### Single-tool (recommended)

One module, one entrypoint, one pixi environment, one tool. The example above. Used today by
`pca-scrapper`, `filtering-r`, the four R modules.

### Stage-language (e.g. `filtering-r`, `pca-python`)

When several tools share **one language and one conda env**, you can dispatch by
parameter inside a single entrypoint:

```yaml
# benchmark side
parameters:
  - method: ["manual", "scrapper_auto"]
```

```
filtering-r/
├── omnibenchmark.yaml      # entrypoints: { filter: run.R }
├── run.R                   # dispatches on args$method
├── src/{manual.R, scrapper_auto.R}
└── envs/filtering-r.yml
```

### Multi-entrypoint (advanced)

One module exposes **multiple entrypoints sharing `src/`**. Only worth it
when several stage entrypoints reuse the same schemas, IO, and env. The
`scanpy` module does this for `pca`, `knn`, `cluster`:

```yaml
entrypoints:
  pca: pca.py
  knn: knn.py
  cluster: cluster.py
  pca-prof: denet pca.py     # values are shell commands — prefix wrappers freely
```

Pick this only when splitting into three repos would force you to vendor or
duplicate shared code, or when you know that a single tool covers multiple stages. The idea is that these modules will outlive the benchmark, and will be useful in many other contexts. But **don't start here.**
