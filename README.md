# Chapter 14: Designing Agentic Systems That Stop Safely

Companion code for *Building Safe Agentic AI for Enterprise Systems* by Mohit Aggarwal.

This repository is the Homecare Visit Triage Agent, a stateful LangGraph evaluation pipeline for document-extraction results from healthcare timesheets. It ingests and normalizes extracted rows, compares them with manually annotated ground truth, checks arithmetic consistency, and pauses for human review when the system cannot safely resolve an ambiguity.

The repository demonstrates a different form of agent safety from the SentinelAI request pipeline. There is no public chat surface to filter. The risks are data loss during normalization, incorrect matching against ground truth, arithmetic inconsistencies, exposure of protected health information (PHI), and automated reporting after a case should have stopped for review.

## What You Will Run

| Chapter section | Demonstration | What it shows |
| --- | --- | --- |
| 14.1 | Denominator-preserving normalization | Rows that fail date or time parsing are counted and reported. They do not disappear before accuracy is calculated. |
| 14.2 | Identity containment | Real filenames and patient identifiers remain in limited in-memory matching paths while persistent artifacts use anonymized identifiers. |
| 14.3 | Human-in-the-loop interruption | Flagged rows pause the LangGraph before reporting. The graph cannot reach the report node until a reviewer supplies a decision. |
| 14.4 | Bounded execution through typed state | The evaluation graph moves typed records between ingestion, normalization, evaluation, triage, human review, and reporting. |
| 14.5 | Comparative extraction evaluation | Six extraction methods are compared against ground truth, with failure analysis and audit artifacts. |

## Production Warning

This repository concerns healthcare-adjacent data and evaluation artifacts. It demonstrates privacy-preserving design patterns; it does not certify HIPAA compliance, establish a business associate agreement, or replace a healthcare organization's privacy, security, retention, access-control, or incident-response program.

The full input package is intentionally excluded from version control. Do not replace it with real patient timesheets, unredacted filenames, or protected data merely to make the benchmark run. Use the approved de-identified study package or a synthetic dataset with the same expected schema.

The `--skip-review` option is for controlled automated testing only. It auto-accepts flagged rows so CI or batch experiments can complete. It must not be used where human review is the actual decision boundary.

## Prerequisites

- Git
- [uv](https://docs.astral.sh/uv/)
- Python 3.11
- An approved, de-identified input package to run the full benchmark or ablation study

No API key, model endpoint, Docker service, or cloud account is required for the test suite. The pipeline evaluates outputs from six extraction methods; it does not call a live vision-language model as part of the benchmark path described here.

## Quick Start

### 1. Install uv

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

### 2. Clone and synchronize the repository

```bash
git clone https://github.com/the-write-path-code/ch14-homecare-visit-triage-agent.git
cd ch14-homecare-visit-triage-agent
uv sync
```

The repository includes `uv.lock`. Its `pyproject.toml` requires Python 3.11. Run `uv sync` after pulling changes so the local environment matches the committed dependency set.

### 3. Run the tests

```bash
uv run pytest
```

This is the correct first run for a fresh clone. The test suite covers the graph behavior without requiring the excluded study inputs.

### 4. Obtain approved benchmark inputs before a full run

The benchmark input directory is intentionally Git-ignored. A full run needs, for each extraction method:

```text
input/<method>/merged_results.xlsx
input/<method>/name_mapping.db
```

It also needs the configured ground-truth workbook, normally referenced by `config.yaml`.

Do not add real patient records to Git. Keep approved data outside the repository where possible, restrict file-system access, and configure the input and output paths through the repository's approved configuration process.

## Configuration

The repository reads its benchmark paths, scoring tolerances, and output locations from `config.yaml`. Keep those values in version control only when they are non-sensitive.

| Configuration area | Purpose |
| --- | --- |
| Input path | Location of each method's extraction output and temporary name-mapping database |
| Ground-truth path | Location of the manually annotated comparison workbook |
| Output path | Directory for generated reports, summaries, and per-method artifacts |
| Hours tolerance | Allowed variance for evaluated hours values |
| Time tolerance | Allowed variance for time-in and time-out values |

The test and benchmark code may load local environment variables, but the exported repository contains no `.env.example`. Do not add credentials, real filenames, patient names, or production paths to committed configuration.

> **Tip**
>
> Run one file with one extraction method before running the full six-method comparison. Confirm the normalized row count, skipped-row count, ground-truth match behavior, flagged rows, and generated artifacts first.

## Run the Chapter Demonstrations

### 1. Run One Benchmark File, Sections 14.1 through 14.4

After placing an approved input package in the configured input location, run one source file through one extraction method:

```bash
uv run python scripts/run_benchmark.py \
  --method band_crop_vlm_cloud \
  --file patient_a_week_1
```

The method name must match an input subdirectory. The file identifier must match a source file represented in that method's `merged_results.xlsx` input.

The run performs these stages:

1. Ingest extraction rows.
2. Normalize dates, times, and fields.
3. Count normalization failures as `normalize_skipped`.
4. Evaluate normalized rows against ground truth.
5. Apply arithmetic and ambiguity checks.
6. Route clean rows toward reporting or flagged rows to human review.

### 2. Test Isolation Across More Than One File

Run the first two available source files for a method:

```bash
uv run python scripts/run_benchmark.py \
  --method band_crop_vlm_cloud \
  --limit 2
```

This is useful for checking that file-specific evaluation state does not leak into the next case.

### 3. Run a Full Method Evaluation

```bash
uv run python scripts/run_benchmark.py \
  --method band_crop_vlm_cloud \
  --all
```

The graph can pause at the human-review node when flagged rows are present. A normal interactive run does not produce a final report until review decisions resume the graph.

### 4. Exercise the Human Review Gate, Section 14.3

When the graph reaches a flagged row, it interrupts at the human-review node. The system returns a review payload containing the flagged records and their evaluation details. An authorized caller must supply a decision through the LangGraph resume path before reporting can continue.

The key behavior is the stop. The graph does not time out into an automatic report, and it does not convert uncertainty into a passing score.

### 5. Run a Controlled Automated Path

For automated tests or controlled batch comparison only, use:

```bash
uv run python scripts/run_benchmark.py \
  --method band_crop_vlm_cloud \
  --all \
  --skip-review
```

This auto-accepts flagged rows and allows the run to finish. It exists for repeatable automation, not for operational decisions involving real records.

### 6. Run All Six Extraction Methods, Section 14.5

```bash
uv run python scripts/run_all_methods.py
```

The batch runner evaluates these configured methods:

```text
band_crop_vlm_cloud
layout_guided_vlm_cloud
layout_guided_vlm_local
ocr_only
ppocr_grid
vlm_full_page
```

It collects the per-method results and writes a unified summary table. The batch path auto-accepts human-review pauses to produce a reproducible comparison snapshot. Treat that behavior as a study-mode exception, not as a production review policy.

### 7. Reproduce the Ablation Study

After the approved inputs are available, run:

```bash
uv run python docs/paper/run_ablation.py
```

The ablation compares naive and protected pipeline behavior, including normalization failures, arithmetic flags, human-review routing, and checks for PHI strings in persistent outputs.

## Expected Results

The pipeline should preserve failure information rather than hide it.

### Normalization

Malformed date or time values are not silently dropped. The pipeline returns both the normalized rows and a skipped-row count. That count remains in the denominator for downstream accuracy calculations.

### Ground-truth evaluation

The evaluation compares extraction output with manually annotated reference data. It distinguishes an inaccurate extraction from an unmatched or ambiguous record. A missing source record cannot be treated as a correct zero-value result.

### Arithmetic checks

The system checks whether reported total hours agree with the duration derived from time-in and time-out values. This check can identify inconsistent output even when a ground-truth label is unavailable or incomplete.

### Human review

Rows flagged for ambiguity, tolerance-edge conditions, or disagreement between extraction status and computed evaluation do not flow directly to reporting. The graph stops until a reviewer decision is provided.

### Persistent artifacts

Reports and logs use anonymized identifiers. Real filenames and names should remain limited to the in-memory matching path required to evaluate an input record against its ground truth.

## Run the Tests

```bash
uv run pytest
```

Run the tests before changing the normalization rules, identity-resolution boundary, tolerance values, graph routing, review interruption, artifact writer, or output schema. The test suite should establish that:

- Parse failures contribute to the skipped-row count.
- Skipped rows remain represented in evaluation denominators.
- Ground-truth matching does not expose real identifiers in persistent artifacts.
- Arithmetic inconsistencies are flagged.
- Flagged rows interrupt the graph before the report node.
- A graph can resume only with an explicit review decision.
- Output files contain anonymized identifiers rather than patient names or source filenames.

## Repository Layout

```text
.
├── README.md
├── pyproject.toml
├── uv.lock
├── config.yaml                         # Paths, tolerances, and non-sensitive settings
├── input/                              # Git-ignored approved study inputs; never commit real PHI
├── output/                             # Generated evaluation artifacts and reports
├── src/
│   ├── config.py                       # Configuration loading
│   ├── models.py                       # Typed extraction, normalized-row, and evaluation state
│   ├── ingestion.py                    # Loads extraction results
│   ├── normalization.py                # Parses and counts normalization failures
│   ├── name_resolver.py                # Limited in-memory identity bridge for ground-truth matching
│   ├── evaluation.py                   # Ground-truth and arithmetic evaluation
│   ├── graph.py                        # LangGraph pipeline and interrupt routing
│   └── reporting.py                    # Anonymized reports and diagnostic artifacts
├── scripts/
│   ├── run_benchmark.py                # One-method benchmark runner
│   └── run_all_methods.py              # Six-method batch comparison
├── docs/paper/
│   ├── run_ablation.py                 # Naive-versus-protected comparison
│   ├── ablation_study.md
│   ├── threat_model.md
│   ├── generalizability.md
│   └── evidence_package.md
└── tests/
```

## Architecture Diagrams and Supporting Documents

The paper documentation records the design and empirical evidence behind the Chapter 14 case study:

- `docs/paper/threat_model.md` maps data-flow risks and controls.
- `docs/paper/ablation_study.md` compares naive and protected pipeline outcomes.
- `docs/paper/evidence_package.md` gathers the benchmark evidence for editorial or research review.
- `docs/paper/generalizability.md` describes which patterns transfer to other document-processing settings.

Read the threat model before changing the name-resolution, reporting, or output-artifact code. The strict boundary is intentional: the name resolver is the only place that handles real filenames and identity mapping.

## Safety and Operational Limits

- This code demonstrates safety patterns for document-extraction evaluation. It does not validate clinical decisions, billing decisions, timesheet approval, or care delivery.
- The full study inputs are deliberately excluded from Git. A public clone cannot reproduce the full empirical result without an approved data package.
- Anonymized identifiers reduce exposure in artifacts. They do not remove the need for access controls, encrypted storage, retention rules, or review of re-identification risk.
- The `--skip-review` mode exists for automated benchmarking. It must not replace human review when the human-review pause is the actual safety boundary.
- The batch runner auto-accepts review holds to create a comparative report. That is appropriate only for a controlled experiment with a documented review policy.
- Keep generated outputs out of public issue threads, unencrypted shared drives, and external tracing tools unless their content has been reviewed and approved.

## Troubleshooting

### A full benchmark run reports missing files

This is expected on a fresh clone. The `input/` directory is Git-ignored. Obtain the approved, de-identified input package and check the paths in `config.yaml`.

### A method name is rejected

The value passed to `--method` must match an input subdirectory exactly. Check the configured input directory before changing the runner.

### The graph pauses and does not write a report

That is the human-review gate working. Inspect the flagged rows and resume the graph with explicit decisions. Do not use `--skip-review` merely to force a report from ambiguous operational data.

### Accuracy looks too high after parsing failures

Inspect the `normalize_skipped` count and the denominator used by the generated report. A metric that excludes rows the system could not parse is not an honest measure of extraction performance.

### A persistent artifact contains a real name or filename

Treat this as a privacy defect. Stop distribution of the artifact, inspect `name_resolver.py` and `reporting.py`, then rerun the artifact scan after correcting the data flow.

### The test suite fails without input files

The unit and graph-behavior tests should not require the excluded study inputs. If a test does, isolate the input-dependent test or provide a synthetic fixture rather than committing real data.

## Related Chapters

- Chapter 6 introduces typed multimodal extraction, confidence handling, PII scanning, and validated handoffs before vector indexing.
- Chapter 7 applies data minimization and deterministic tool boundaries to sensitive home-healthcare staffing context.
- Chapter 12 establishes idempotency when repeated delivery could cause duplicate writes.
- Chapter 13 adds optimistic concurrency control when state changes during a decision or approval interval.
- Chapter 15 turns human overrides, adversarial cases, regression baselines, and fail-closed behavior into repeatable CI checks.

## License and Errata

See `LICENSE` for licensing terms. Report documentation or code issues through this repository's GitHub issue tracker. Do not include protected data, real filenames, credentials, or unredacted artifacts in an issue report.
