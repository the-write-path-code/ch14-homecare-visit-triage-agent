# Homecare Visit Triage Agent — Architecture

This file is the companion architecture note for Chapter 14.
It describes the current evaluation graph, not the original build plan.

## 1. What this system is

The Homecare Visit Triage Agent is a stateful evaluation pipeline built with LangGraph to benchmark document extraction outputs across healthcare timesheets. It does not perform optical character recognition (OCR) or vision-language model (VLM) extraction directly; rather, it ingests pre-extracted timesheet rows produced by upstream extraction methods. The pipeline normalizes raw row entries into typed schema models, evaluates them deterministically against labeled ground truth data, and calculates per-field accuracy and arithmetic consistency. When ambiguous entries or borderline discrepancies arise, the pipeline halts execution at a Human-in-the-Loop (HITL) boundary using an interrupt mechanism to collect reviewer resolutions. Once human decisions are incorporated or if no review is required, the pipeline generates aggregated, anonymized benchmark artifacts. The codebase is organized across dedicated modules: `src/nodes.py` (node functions), `src/graph.py` (graph topology and compilation), `src/evaluation.py` (scoring and row evaluation), `src/name_resolver.py` (ephemeral identity resolution), `src/normalization.py` (type conversion and schema enforcement), and `src/reporting.py` (artifact generation).

## 2. Graph topology

The pipeline is modeled as a compiled `StateGraph` operating over `BenchmarkState` with five sequential and conditional nodes:
- `ingest` (`ingest_node` in `src/nodes.py`): Ingests extracted records from the input spreadsheet into raw `ExtractionRow` structures.
- `normalize` (`normalize_node` in `src/nodes.py`): Validates and converts raw string fields into strongly-typed `NormalizedRow` objects via `src/normalization.py`. Any unparseable or malformed rows are counted in `normalize_skipped` to maintain denominator integrity for downstream scoring.
- `evaluate` (`evaluate_node` in `src/nodes.py`): Evaluates normalized rows against ground truth via `evaluate_file()` in `src/evaluation.py`, computing per-field matches (hours, time-in, time-out) within configured tolerances and identifying rows requiring human inspection.
- `human_review` (`human_review_node` in `src/nodes.py`): Suspends graph execution at the HITL boundary using LangGraph's `interrupt()` call, serializing the flagged rows and waiting for an external resume command containing reviewer decisions.
- `report` (`report_node` in `src/nodes.py`): Aggregates file and method results via `src/reporting.py` and writes anonymized summary artifacts (`run_summary.json`, `paper_table.md`, `failures.json`, and per-method evaluation files).

### Graph Wiring and Triage Rule

Linear edges connect `START -> ingest -> normalize -> evaluate`. After the `evaluate` node completes, a conditional edge invokes `triage_decision`:
- If `needs_review` is `True` and `flagged_for_review` contains flagged rows, the graph routes to `human_review`.
- Otherwise, the graph routes directly to `report`.

Graph compilation in `src/graph.py` configures `interrupt_before=["human_review"]`. When human review is triggered, `report` cannot execute until the graph state is explicitly resumed with human review decisions.

### Review Triggers

Review gating is evaluated during row scoring in `src/evaluation.py`:
- **Hours mismatch within tolerance:** A row where the hours difference is within the acceptable window but exceeds two-thirds of the tolerance limit (borderline hours, e.g., > 10 minutes for a ±15 minute tolerance window).
- **Multiple GT rows for the same (file, date):** Ambiguous grounding where multiple ground truth records match the same file and service date. (In the current codebase, `NameResolver` establishes deterministic 1-to-1 sequential mapping to resolve unique GT keys, while the evaluation engine flags ungrounded coverage gaps).
- **Status is `"flagged"` but values match GT:** A row flagged by the upstream extraction validator whose computed hours nonetheless match ground truth, indicating a disagreement between extractor heuristics and ground truth annotations.

## 3. PHI boundary

Protected Health Information (PHI) isolation is enforced structurally through module boundaries:
- `src/name_resolver.py` is the only module in the entire codebase that handles real patient names and real source filenames. It connects to `name_mapping.db` in-memory.
- In `src/evaluation.py`, `evaluate_file()` invokes `NameResolver.resolve_for_date()` / `NameResolver.resolve()` strictly in-memory to look up ground truth records in the ground truth dictionary. Once the dictionary lookup completes, the real filename is discarded.
- `NormalizedRow.source_file`, `RowEvalResult.source_file`, and `FileEvalResult.source_file` strictly store anonymized identifiers (e.g., `patient_a_week1.pdf`).
- Real patient names and real filenames are never persisted in state, logged, or written to output artifacts (`run_summary.json`, `failures.json`, `paper_table.md`).

## 4. Diagrams used in Chapter 14

### LangGraph state machine

<!-- Printed as Chapter 14 Figure 14.4. Do not edit this mermaid. -->

```mermaid
%%{init: {"theme": "base", "themeVariables": {"fontFamily": "Arial, Helvetica, sans-serif", "fontSize": "24px", "primaryColor": "#F8FAFC", "primaryBorderColor": "#0284C7", "primaryTextColor": "#000000", "lineColor": "#475569"}}}%%
flowchart TD
    IngressNorm["<div style='min-width: 820px;'><b>1. Batch Ingestion &amp; Schema Normalization</b><br/><code>IngestNode</code> reads spreadsheet into raw rows; <code>NormalizeNode</code> enforces date/time typing and tallies malformed rows</div>"]

    Evaluate["<div style='min-width: 820px;'><b>2. EvaluateNode (Deterministic Ground Truth Scoring)</b><br/>Scores fields against ground truth with &plusmn;15m/&plusmn;30m tolerances; flags ambiguous entries</div>"]
    IngestNorm -->|"Normalized Rows"| Evaluate

    Triage["<div style='min-width: 820px;'><b>3. TriageDecision Gate (Evaluation &amp; Review Trigger)</b><br/>Checks: borderline hours (&gt;10m) &bull; ambiguous multi-GT candidates &bull; extractor flag with math match</div>"]
    Evaluate -->|"EvalResults &amp; Flags"| Triage

    Human["<div style='min-width: 360px;'><b>HumanReviewNode</b><br/><b>LangGraph interrupt()</b><br/>Suspends graph for reviewer resolution</div>"]
    Clean["<div style='min-width: 360px;'><b>Automated Direct Path</b><br/>Zero ambiguities detected;<br/>immediate throughput</div>"]

    Triage -->|"needs_review == True"| Human
    Triage -->|"needs_review == False"| Clean

    Report["<div style='min-width: 820px;'><b>4. ReportNode &amp; Final Benchmark Publication</b><br/>Aggregates file metrics, reconciles reviewer inputs &amp; writes <code>paper_table.md</code>, <code>run_summary.json</code></div>"]

    Human -->|"Resume with Decisions"| Report
    Clean --> Report

    classDef node fill:#EBF5FF,stroke:#2563EB,color:#000000,stroke-width:1.5px
    classDef triage fill:#FEF9C3,stroke:#CA8A04,color:#000000,stroke-width:2px
    classDef hitl fill:#FEE2E2,stroke:#DC2626,color:#000000,stroke-width:2px
    classDef clean fill:#DCFCE7,stroke:#15803D,color:#000000,stroke-width:1.5px
    classDef report fill:#EDE9FE,stroke:#7C3AED,color:#000000,stroke-width:1.5px

    class IngestNorm,Evaluate node
    class Triage triage
    class Human hitl
    class Clean clean
    class Report report
```

### Real-filename isolation

<!-- Printed as Chapter 14 Figure 14.5. Do not edit this mermaid. -->

```mermaid
%%{init: {"theme": "base", "themeVariables": {"fontFamily": "Arial, Helvetica, sans-serif", "fontSize": "24px", "primaryColor": "#F8FAFC", "primaryBorderColor": "#0284C7", "primaryTextColor": "#000000", "lineColor": "#475569"}}}%%
flowchart TD
    Ingress["<div style='min-width: 880px;'><b>1. Untrusted Extraction Output</b><br/><code>Extracted Row</code> (carrying <code>source_file: patient_c_week3.pdf</code>) &rarr; <code>NormalizeNode</code> validates typing</div>"]

    Resolver["<div style='min-width: 410px;'><b>Name Resolver (Secure Memory)</b><br/>Queries <code>name_mapping.db</code> in-memory<br/>Maps <code>patient_c</code> &rarr; <code>J. Doe</code></div>"]
    GTMap["<div style='min-width: 410px;'><b>Ground Truth Map (Secure Memory)</b><br/>Loads <code>ground_truth.xlsx</code><br/>Indexes lookup key: (J. Doe, Date)</div>"]

    Ingress -->|"Queries: patient_c"| Resolver
    Resolver -->|"Lookup Key"| GTMap

    Eval["<div style='min-width: 880px;'><b>2. EvaluateNode (Deterministic Ground Truth Scoring &amp; PHI Isolation)</b><br/>Compares normalized row against GT row within &plusmn;15m/&plusmn;30m tolerances</div>"]
    Ingress -->|"Anon Row"| Eval
    GTMap -->|"Yields GT Row"| Eval

    Result["<div style='min-width: 880px;'><b>3. RowEvalResult (Pydantic Model Boundary)</b><br/>Real name discarded in-memory; output fields strictly retain <code>source_file: patient_c_week3.pdf</code></div>"]
    Eval -->|"Matches Math"| Result

    Summary["<div style='min-width: 410px;'><b>run_summary.json (Public Domain)</b><br/>Strictly <code>patient_c_week3.pdf</code></div>"]
    Failures["<div style='min-width: 410px;'><b>failures.json (Public Domain)</b><br/>Strictly <code>patient_c_week3.pdf</code></div>"]

    Result -->|"Strictly Anonymized"| Summary
    Result -->|"Strictly Anonymized"| Failures

    classDef untrusted fill:#FEF9C3,stroke:#CA8A04,color:#000000,stroke-width:1.5px
    classDef secure fill:#FFF1F2,stroke:#E11D48,color:#000000,stroke-width:1.5px
    classDef eval fill:#E0F2FE,stroke:#0284C7,color:#000000,stroke-width:1.5px
    classDef result fill:#EDE9FE,stroke:#7C3AED,color:#000000,stroke-width:1.5px
    classDef public fill:#DCFCE7,stroke:#15803D,color:#000000,stroke-width:1.5px

    class Ingress untrusted
    class Resolver,GTMap secure
    class Eval eval
    class Result result
    class Summary,Failures public
```
