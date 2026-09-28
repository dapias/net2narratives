# net2narratives

Evaluating LLM-generated interpretations of cross-sectional psychological
networks.

`net2narratives` takes an already-estimated cross-sectional network (nodes,
edges, weights, community and centrality information) and asks an LLM to
interpret it in natural language, following a reporting-constrained
protocol grounded in psychological-network methodology. How the network was
estimated (which data, which correlation method, which nodes) is the job of
an upstream pipeline; `net2narratives` only needs a JSON file in the format
described in [`schema/network_for_llm.md`](schema/network_for_llm.md).

The repository also contains the validation suite used in the accompanying
manuscript: hand-built and procedurally generated networks with known answers,
and a deterministic, rule-based scorer for six competencies (edge-sign
fidelity, unsupported-association avoidance, magnitude-label consistency,
community-membership grounding, centrality interpretation, and
causal-language avoidance).

## Accompanying manuscript

Tapias, D. (2026). *From Networks to Narratives: Evaluating LLM-Based
Interpretation of Psychological Networks.* The results reported in the
manuscript were produced with release
[v1.0.5](https://github.com/dapias/net2narratives/releases/tag/v1.0.5).

**Main finding.** High performance on isolated diagnostic tasks did not
guarantee faithful full-length reports. On the controlled and procedural
benchmarks, where each network probes one competency, performance under
the full protocol was close to ceiling for most competencies. When the same protocol
was applied to two empirical psychological networks, the full-length
reports still contained errors of the kinds the benchmarks were designed to
detect. Benchmark competence should therefore be read as a necessary check,
not as evidence that an LLM-generated network report can be used without
expert review.

## Install

```bash
git clone https://github.com/dapias/net2narratives.git
cd net2narratives
pip install -e .
```

Requires Python >= 3.9. R is needed only to rebuild the two empirical
networks under `empirical_networks/`; the package and the validation suite
are pure Python.

## Quickstart

1. **Set your API key.** Any OpenAI-compatible chat-completions endpoint
   works. Export the variable named in your config, or put it in a `.env`
   file in your working directory (never commit this file):

   ```bash
   export GWDG_API_KEY=...   # or whichever variable your config names
   ```

2. **Copy the example LLM config** and adjust `base_url` / `model` if you
   are not using GWDG's Academic Cloud:

   ```bash
   cp examples/llm_config.example.yaml my_llm_config.yaml
   ```

3. **Interpret the bundled example network** (a small hand-built network
   with two communities and one weak bridge):

   ```bash
   net2narratives-interpret --network examples/network_for_llm.example.json \
                            --llm-config my_llm_config.yaml
   ```

   This writes `examples/network_for_llm.example_interpretation.md` (the
   model's response) and a matching `.audit.json` recording the exact
   system and user prompt, model, and timestamp.

4. **Interpret your own network** by supplying a JSON file in the schema
   described in [`schema/network_for_llm.md`](schema/network_for_llm.md).
   `examples/network_for_llm.example.json` and
   `net2narratives/validation/synthetic_networks.py` contain worked
   examples. Edges are sorted by descending absolute weight before being
   sent to the model.

## Prompt templates

Three prompt templates are bundled in `net2narratives/prompts/`, in order
of increasing methodological scaffolding:

- `naive.md`: minimal context, general-audience interpretation, no
  grounding or methodological constraints.
- `scientific_minimal.md`: basic source-grounding instruction and a
  cross-sectional causal-language caution, for a research audience.
- `full_protocol.md`: the full reporting-constrained protocol used for the
  paper's main results (magnitude-label convention, absent-edge caveats,
  network-relative centrality interpretation, sign handling, and a
  plain-language plus technical output format).

Select one with `prompt_template` in your LLM config, or give the path to
your own template (it must contain `## System prompt` and `## User prompt`
headings, each on its own line).

## Validation suite

```bash
net2narratives-validate --llm-config my_llm_config.yaml --out-dir results_synthetic
```

This runs the six hand-built diagnostic networks
(`net2narratives/validation/synthetic_networks.py`) against the configured
model and scores each response with the deterministic evaluator
(`net2narratives/validation/scorer.py`). Useful options:

```bash
net2narratives-validate --llm-config my_llm_config.yaml --repeats 50 --out-dir results_synthetic    # repeated generations
net2narratives-validate --llm-config my_llm_config.yaml --test-id calibration --out-dir results_test  # a single test
net2narratives-validate --llm-config my_llm_config.yaml --source procedural \
    --n-per-competency 21 --proc-seed 4242 --out-dir results_procedural                           # procedural networks
```

Each run writes the raw model output and a score file per test, plus a
`summary.json` in the output directory (`--out-dir`). Each result records
the model, a UTC timestamp, the network source (hand-built or procedural),
and, for procedural networks, their generation parameters. Interrupted runs
resume from `summary.json` when rerun with the same `--out-dir`.

The tables in the paper report, for each model and competency, the
proportion of passed runs and the mean score computed from these
`summary.json` files.

### Reproducing the paper's benchmarks

The configurations used for the paper are in `configs/`:

| Config | Used for |
|---|---|
| `llm_config_gptoss.yaml` | reference model (`gpt-oss-120b`), full protocol |
| `llm_config_glm.yaml` | `glm-4.7`, full protocol |
| `llm_config_qwen.yaml` | `qwen3.5-397b-a17b`, full protocol |
| `llm_naive.yaml` | reference model, naive prompt |
| `llm_scientific_minimal.yaml` | reference model, scientific-minimal prompt |

Procedural networks are generated deterministically from
`--n-per-competency` and `--proc-seed`, so they are not stored in the
repository. The paper uses 21 networks per competency (126 procedural
networks; together with the six hand-built networks, 132 in total) and
seed 4242. Both values are needed: the generator draws a different set of
networks for a different `--n-per-competency`, so the paper's networks are
not the first 21 of a larger battery.

```bash
net2narratives-validate --llm-config configs/llm_config_gptoss.yaml \
    --source procedural --n-per-competency 21 --proc-seed 4242 \
    --out-dir results/procedural_gptoss
```

Without these options, `--source procedural` uses a smaller default
battery (10 networks per competency, seed 2026) intended for quick local
checks. To inspect the generated networks without calling an LLM:

```bash
python3 -m net2narratives.validation.procedural_networks \
    --n-per-competency 21 --seed 4242 --out procedural_networks_seed4242.json
```

### Template baseline

A deterministic, non-LLM template baseline renders statements from the
supplied network alone (without access to the ground truth) and scores them
with the same evaluator. It needs no API key and should pass every network,
confirming that a perfect score is attainable:

```bash
python3 -m net2narratives.validation.baselines --source both \
    --n-per-competency 21 --seed 4242 --out-dir baseline_results
```

## Empirical case studies

`empirical_networks/` contains the two empirical networks audited in the
paper. Each subfolder contains the R script that builds a
`network_for_llm.json` file from the source data, the resulting JSON file
used in the paper, a network figure, and a `sessionInfo_*.txt` file
recording the R and package versions. See
[`empirical_networks/README.md`](empirical_networks/README.md) for
provenance and reproduction steps.

- **`work_strain_engagement/`**: the Work Strain and Engagement network,
  re-estimated from the data and analysis code of Bereznowski, Atroszko &
  Konarski (2023). The source data are not redistributed here; download
  `dataset.csv` from the authors' OSF deposit (<https://osf.io/jvqfa/>) and
  place it in `data/` to rebuild the network. The resulting
  `network_for_llm.json` used in the paper is included.
- **`personality/`**: the Personality network, estimated from the 25-item
  `bfi` dataset distributed with the R packages `psych`/`psychTools`; no
  separate download is needed.

## Repository layout

```
net2narratives/
  interpret.py              # net2narratives-interpret: single-network interpretation
  prompts/
    full_protocol.md        # full reporting-constrained protocol (main results)
    naive.md                # naive comparison prompt
    scientific_minimal.md   # scientific-minimal comparison prompt
  validation/
    synthetic_networks.py   # 6 hand-built diagnostic networks, one per competency
    procedural_networks.py  # procedurally generated networks (structural variation)
    scorer.py               # deterministic, rule-based evaluator
    run_validation.py       # net2narratives-validate: runs and scores the suite
    baselines.py            # deterministic template baseline (no API needed)
schema/
  network_for_llm.md        # input format
configs/                    # LLM configurations used for the paper
examples/
  llm_config.example.yaml
  network_for_llm.example.json
empirical_networks/
  README.md
  work_strain_engagement/
    export_bereznowski_network.R
    data/                   # place dataset.csv from OSF here (not tracked)
    network_for_llm.json
    occupational_wellbeing_network.pdf
    sessionInfo_bereznowski.txt
  personality/
    export_bfi_network.R
    network_for_llm.json
    bfi_personality_network.pdf
    sessionInfo_bfi.txt
pyproject.toml
LICENSE
```

## Citation

If you use this package, please cite:

> Tapias, D. (2026). From Networks to Narratives: Evaluating LLM-Based
> Interpretation of Psychological Networks.

## License

MIT
