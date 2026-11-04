<div align="center">

# 🧬 NanoRemix

**RNA modification prediction for RNA002 and RNA004**

Binary & Multi · Selected modifications · Third-generation & fused outputs

[Quick Start](#quick-start) · [Inputs](#inputs) · [Outputs](#outputs) · [Models](docs/MODELS.md) · [中文](docs/README.zh-CN.md)

**v0.3.0** · Binary & Multi pretrained model bundle · Python 3.10/3.11

</div>

> **Download:** the source repository and its ZIP do not contain large weights.
> Download only the model asset ZIPs required for your chemistry, task, strategy,
> and fusion route from the matching GitHub Release. Four third-generation ZIPs
> separate RNA002/RNA004 Binary/Multi; one shared ZIP contains all 25 second-generation
> checkpoints. This keeps each upload smaller and avoids duplicate research archives.
> See [model installation](docs/MODELS.md). No retraining is required.

## 🧩 Function

NanoRemix predicts RNA modifications from prepared sequence-signal features or
existing model probabilities. Choose the chemistry, task and modifications;
unrequested modifications do not compete in the final prediction.
Matching unmodified bases remain as controls.

| Your data | Default output | Optional outputs |
| --- | --- | --- |
| Biological | **Second + third generation fused** | `--also-thirdgen`, `--also-secondgen` |
| Synthetic | **Third generation only** | No second-generation input needed |

Second/third generation refers to **NanoRemix model versions**, not sequencing
technologies. The second generation uses sequence evidence; the third uses
chemistry-specific sequence-signal evidence. Biological runs require both sources
and do not silently fall back to third-generation-only prediction.

| Choice | Options |
| --- | --- |
| Chemistry | `RNA002`, `RNA004` |
| Task | `binary`: modification versus canonical base; `multi`: selected classes within the reference-base family |
| Modifications | `--mods m6A,m5C,pseudoU`, another supported subset, or `all` |
| Strategy | `s1`: independent shifted views; `s2` (default): joint multi-position model; `both`: separate S1/S2 results |
| Output filtering | All observations, accepted observations, or accepted modified observations |

S1's positional-model combination is not second-plus-third fusion.
In extended fused output, classes unsupported by the second generation retain
third-generation evidence and are identified in the output.

<a name="quick-start"></a>

## 🚀 Quick Start

From the repository root:

```bash
python -m venv .venv
source .venv/bin/activate
python -m pip install -e .

nanoremix predict \
  --input examples/minimal_binary_probabilities.csv \
  --task binary --chemistry RNA004 --mods m6A \
  --input-kind probabilities --data-type synthetic --strategy s2 \
  --output results/quickstart
```

On Windows PowerShell, activate with `.venv\Scripts\Activate.ps1`.
The bundled example contains **artificial probabilities**, needs no weights,
and demonstrates software behavior, not biological accuracy.

| record_key | predicted_label | prob_A | prob_m6A | confidence |
| --- | --- | --- | --- | --- |
| demo-001 | m6A | 0.10 | 0.90 | 0.90 |
| demo-002 | A | 0.80 | 0.20 | 0.80 |

Use a new output directory for each run.

<a name="inputs"></a>

## 📥 Inputs

Supply a UTF-8 CSV/TSV, optionally gzip-compressed. **One row is one candidate
observation**, such as a read covering a target site.

| Input | Required contents |
| --- | --- |
| Prepared features (default) | `9mer_final` plus signal descriptors; matching third-generation weights |
| Binary probabilities | `center_ref_base` and `s1_prob_<mod>` / `s2_prob_<mod>` for the requested modifications and strategies |
| Multi probabilities | `center_ref_base` and the complete native class-probability axis for each requested strategy |
| Biological data, additional source | All 15 second-generation class probabilities, or centered 81-nt sequences plus the second-generation model |

For feature input, `9mer_final` contains **9 nt for RNA002** or **17 nt for
RNA004**. Signals are `mean_output1..K`, `median_output1..K`, `std_output1..K`,
and `length_output1..K`, where K = 5 or 9, respectively.
Values must use the original preprocessing and normalization.
This package does **not** basecall or extract features directly from POD5/FAST5.

Use `record_key` for observation identity; `sample_id`, `read_id`, `site_id` and
reference coordinates are optional metadata. Separate second-generation tables
are joined by explicit identity keys, not row order.
See [input schemas and FASTA options](docs/INPUTS.md).

## ⚙️ Predict With Models

Use Python 3.10/3.11 with the original-compatible model dependencies:

```bash
python -m pip install -e '.[models,second]' -c constraints-models.txt
nanoremix doctor --model-root models --model-checksums models/SHA256.json --strict
```

Download the matching model asset ZIP or ZIPs from the GitHub Release and extract
each into the repository root to create/extend `models/`. A pure third-generation
run needs one chemistry/task ZIP. Sequence-based fusion additionally needs the
shared second-generation ZIP. The checker lists missing files and checks manifest
metadata and hashes without executing models. Passing this check is not a
real-model prediction test.

**Biological data**, with a prepared feature table and a `sequence81` column:

```bash
nanoremix predict \
  --input prepared004.csv --task multi --chemistry RNA004 \
  --mods m6A,m1A,m5C,pseudoU --data-type biological --strategy s2 \
  --sequence-column sequence81 \
  --model-root models --model-checksums models/SHA256.json --trust-model-files \
  --output results/biological
```

Add `--also-thirdgen --also-secondgen` to save both component outputs.
Alternatively supply `--second-probabilities second.tsv --join-on record_key`
instead of `--sequence-column`; this route does not need second-generation weights.

**Synthetic data**, with third-generation weights only:

```bash
nanoremix predict \
  --input prepared002.csv --task binary --chemistry RNA002 \
  --mods m6A,pseudoU --data-type synthetic --strategy s2 \
  --model-root models --model-checksums models/SHA256.json --trust-model-files \
  --output results/synthetic
```

These commands require your own prepared data. To check model execution with
artificial feature inputs, run `python examples/make_feature_demo.py --output demo-features`
and use `demo-features/RNA002_features.csv` or `RNA004_features.csv` as input.
That check is not a biological benchmark. Load models only from a trusted source.

<a name="outputs"></a>

## 📤 Outputs

| File | Contents |
| --- | --- |
| `fused/s2/predictions.tsv` | Biological fused predictions by default |
| `thirdgen/s2/predictions.tsv` | Synthetic predictions, or optional biological third-generation output |
| `secondgen/none/predictions.tsv` | Optional biological second-generation output, saved once |
| `selection_audit.tsv` beside each prediction table | Every input observation and any rejection reason |
| `run_manifest.json` | Options, selected labels, input/model hashes and observation counts |

With `--strategy s1` or `both`, corresponding `s1/` directories are written.
Prediction columns include identity, `predicted_label`, selected `prob_<label>`
columns, `confidence`, `status` and `evidence_source`.
Rejected calls have no accepted label.

Probabilities are conditional on the selected classes and known reference-base
family. Confidence is the winning-class probability, not necessarily modification
probability. Omitted true modifications may be assigned to a selected class.
The tool does not automatically combine observations into site-level calls or
modification fractions. Second-only output cannot predict unsupported classes.

## 📚 Reference

| Topic | Reference |
| --- | --- |
| Input columns and sequence alignment | [Inputs](docs/INPUTS.md) |
| Required weights, calibration and model layout | [Models](docs/MODELS.md) |
| Supported modifications and additional options | [Options](docs/OPTIONS.md) |
| 中文功能与使用说明 | [中文](docs/README.zh-CN.md) |
| Model-file safety | [Security](SECURITY.md) |

Use `nanoremix catalog` for supported labels and `nanoremix predict --help` for
all options. This is an inference package, not an automatic reproduction of all
manuscript figures and tables. The source [license](LICENSE) requires rights-holder
confirmation before public redistribution.
