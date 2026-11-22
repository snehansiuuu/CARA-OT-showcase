# CARA-OT

**Cross-Artifact Change Impact and Recovery Assurance for Operational Technology**

When one maintenance visit changes a PLC program, an HMI alarm and a firewall rule within minutes, existing tools report three changed files. CARA-OT reports **one chain of related changes**: what exactly changed, which equipment it can affect, whether anyone approved it, and whether the plant can be restored from its backups. A local LLM then explains the evidence in plain English, and the asset IDs and measured values it writes are checked against that evidence.

![Python 3.13](https://img.shields.io/badge/Python-3.13-3776AB?logo=python&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-009688?logo=fastapi&logoColor=white)
![NetworkX](https://img.shields.io/badge/NetworkX-dependency%20graph-4C72B0)
![Ollama](https://img.shields.io/badge/Ollama-llama3.2%3A3b%20(local)-000000)

> **This is a showcase repository.** It contains the project description, architecture, screenshots, results and selected code excerpts. The full source code is kept in a private repository and is available to recruiters and examiners on request.

![Animated walkthrough of the CARA-OT dashboard](docs/demo.gif)

## The problem

OT environments (PLCs, HMI/SCADA, engineering workstations, industrial firewalls, drives) change constantly through maintenance and engineering work. A file-hash or text-diff tool can say *that* a file changed. It cannot say which parameter moved, that a firewall change and a PLC change minutes later touch the same controller, which pump sits downstream, whether a ticket approved the work, or whether a verified backup exists to roll it back.

## What CARA-OT answers

The reference scenario, **WTR-S1** on a simulated water pumping station, combines three changes made between 23:12 and 23:26:

| Question | CARA-OT's answer for WTR-S1 |
|---|---|
| What exactly changed? | Firewall FW-01 now allows EWS-01 → PLC-02 (tcp/502 Modbus, tcp/8080 OpenPLC programming); PLC-02 protective timer `trip_tmr.PT` 2 s → 8 s; HMI alarm `ALM-PT02-HH` disabled |
| Do the changes belong together? | Yes: each one reaches PLC-02, 7 minutes after the previous one, so they form one chain |
| What could be affected? | Pump-02 and Process-Line-01, each with the dependency-graph path that reaches it |
| Was it approved? | No: no maintenance ticket covers any of the three changes |
| Can it be recovered? | Not fully: 3 of 4 required backup components verified; the HMI-01 project backup is missing |
| How serious is it? | CRITICAL, risk score 18: new access path → protection weakened → operator visibility removed |

## Architecture

```mermaid
flowchart TB
  subgraph S1["1 · Read and normalise"]
    direction LR
    A["Exported OT files<br/>PLC · HMI · firewall<br/>drives · backups · tickets"] --> B["Ingest<br/>SHA-256 + time"] --> C["Parser plugins<br/>one per format"] --> D["Semantic diff<br/>baseline vs current"] --> E["OT dependency<br/>graph"]
  end
  subgraph S2["2 · Deterministic analysis"]
    direction LR
    F["Impact<br/>graph traversal"] --> G["Authorisation<br/>vs tickets"] --> H["Correlation<br/>shared equipment"] --> I["Recovery<br/>assurance"] --> J["Risk<br/>score"]
  end
  subgraph S3["3 · Evidence and explanation"]
    direction LR
    K["Evidence<br/>bundle"] --> L["Local LLM<br/>explanation"] --> M["Grounding check<br/>IDs and values"] --> N["Analyst<br/>dashboard"]
  end
  S1 --> S2 --> S3
```

Three design decisions shape the system:

- **Deterministic evidence; the AI only explains.** Every finding comes from the engines above. The LLM only rewrites the evidence bundle in plain English, and a checker verifies that the asset IDs and measured values (numbers with units) it writes appear in that bundle. If the model is unavailable, a template explanation is used instead.
- **Read-only by design.** CARA-OT analyses exported files and has no write path to PLCs, HMIs or firewalls.
- **Plugin parsers, one model.** Each file format is a parser plugin that emits the same entities and typed relations, so the analysis engines never see vendor formats.

<details>
<summary><b>Worked example:</b> the pipeline with WTR-S1's measured values</summary>
<br>
<img src="docs/figures/fig_paper_pipeline.png" width="420" alt="CARA-OT pipeline with the measured values of scenario WTR-S1">
</details>

## Demo

The dashboard runs locally (FastAPI back end, plain HTML/CSS/JavaScript front end). Scenario event times in the screenshots are simulated.

**Water pumping station, WTR-S1.** One CRITICAL chain of three unauthorised changes, with the graph path to the affected pump.

![Dashboard overview for scenario WTR-S1](docs/screenshots/01_overview_water_WTR-S1.png)

**Chain detail.** Why the changes are linked, potential impact, recovery assurance and the transparent risk score.

![Chain detail for WTR-S1](docs/screenshots/02_chain_detail_WTR-S1.png)

**Packaging line, MFG-M1.** The same engines on a second plant: four unauthorised changes across firewall, PLC, drive and HMI form one chain.

![Dashboard overview for scenario MFG-M1](docs/screenshots/03_overview_manufacturing_MFG-M1.png)

**Recovery assurance, WTR-S3.** Each backup is checked for presence, SHA-256 integrity and content/version; a missing HMI project, a missing PLC program and a corrupted firmware image leave 3 of 6 systems fully recoverable.

![Recovery assurance table for WTR-S3](docs/screenshots/06_recovery_assurance_WTR-S3.png)

**Local LLM explanation.** llama3.2:3b through Ollama on a laptop CPU (14.6 s); 12 of 12 checked identifiers and values are supported by the evidence.

![Grounded explanation for WTR-S1](docs/screenshots/11_ai_explanation_WTR-S1.png)

<details>
<summary><b>More screenshots:</b> noise control, unrelated changes, approved maintenance, scenario builder, live-folder mode</summary>
<br>
<p><b>Cosmetic edits ignored (MFG-M3).</b> 3 files and 8 text lines changed, but only 1 semantic change; 2 files hold cosmetic-only edits.</p>
<img src="docs/screenshots/05_cosmetic_edits_ignored_MFG-M3.png" alt="Raw diff versus semantic changes for MFG-M3">
<p><b>Unrelated changes kept apart (WTR-S4).</b> Two changes on different equipment stay separate, and a firewall rule re-order with the same effect is not reported.</p>
<img src="docs/screenshots/04_unrelated_changes_kept_separate_WTR-S4.png" alt="Two independent changes in WTR-S4">
<p><b>Approved maintenance (WTR-S2).</b> A ticketed setpoint change with a fresh backup is assessed as low risk.</p>
<img src="docs/screenshots/07_authorised_maintenance_low_risk_WTR-S2.png" alt="Authorised maintenance in WTR-S2">
<p><b>Scenario builder.</b> Changes are picked from drop-downs built from the plant's own files; every run starts from a clean copy of the baseline.</p>
<img src="docs/screenshots/08_scenario_builder.png" alt="Scenario builder">
<p><b>Live-folder mode.</b> A hand edit to a PLC file (motor_temp_limit 80 → 95 °C) is reported as one unauthorised, safety-related change.</p>
<img src="docs/screenshots/09_live_folder_hand_edit.png" alt="Live-folder mode detecting a hand edit">
</details>

## Results

CARA-OT (E) was compared with four simpler approaches on 12 scenarios across two simulated plants: 26 labelled changes, 5 cosmetic-only edits and 7 true change chains.

![In-sample comparison of approaches A to E](docs/figures/fig_comparison_chart.png)

CARA-OT found 26 of 26 changes with 0 false alarms and all 7 chains. The closest baseline, keyword and regex rules, reached 96% detection F1 with 2 false alarms but only 68% affected-asset F1. A standalone LLM given the raw diffs reached 43% detection F1 with 54 false alarms.

> **How to read these numbers.** The 12 scenarios were written while CARA-OT was being built and were used to tune it, so these are in-sample results. They show that the design works as intended, not general accuracy. An independently written, held-out scenario set is the next step.

| Engineering measure | Result |
|---|---|
| Automated tests (pytest) | 28 passed |
| Functional test cases TC-01 to TC-32 | 25 passed (6 on renamed testbed assets), 4 partial, 1 deviation, 1 not implemented, 1 not executed |
| Analysis time | 69 ms mean per scenario over 60 timed runs |
| Scale test | Generated 203-asset environment analysed in 1.1 s |
| Local LLM explanation | 8–15 s on a laptop CPU |
| Code size | ≈ 3,400 lines of Python in the engine, ≈ 830 lines of tests and evaluation, ≈ 890 lines of dashboard |

## Tech stack

| Area | Technology |
|---|---|
| Language | Python 3.13 |
| Web API | FastAPI, Uvicorn |
| Dependency graph | NetworkX |
| Data files | PyYAML, JSON, CSV |
| Local AI | Ollama with llama3.2:3b |
| Dashboard | HTML, CSS and JavaScript, no framework |
| Testing | pytest |
| OT inputs analysed | IEC 61131-3 Structured Text (OpenPLC-compatible), HMI JSON exports, iptables rules, drive parameter CSV, YAML inventory, backup catalogue and maintenance tickets |

## Selected code

[CODE_SNIPPETS.md](CODE_SNIPPETS.md) shows short, unmodified excerpts from the private source: the parser plugin interface, the semantic diff, impact propagation, change correlation, recovery checks and the LLM grounding checker.

## Limitations and next steps

- The two plants are simulated sets of exported files. The PLC programs are OpenPLC-compatible Structured Text but were not run on a PLC runtime.
- The evaluation is in-sample, on self-authored scenarios; there is no held-out set yet.
- Parser coverage is limited to a Structured Text subset, a generic HMI JSON export and iptables TCP rules; PLCopen XML and vendor project formats are not supported.
- Role-based access and report export are not implemented.
- The grounding check verifies identifiers and numbers, not every interpretation in the explanation.

Next: held-out scenarios written by someone who has not seen the engine, an OpenPLC runtime testbed, a PLCopen XML parser and report export.

## About

**Snehansu Nag**, B.Tech in Computer Science and Engineering (Cyber Security), SRM Institute of Science and Technology. Final-year major project, 2026.

The full source code, test evidence and a live demo are available on request; contact me through my GitHub profile.

© 2026 Snehansu Nag. All rights reserved. See [LICENSE](LICENSE).
