# AGENTS.md — Spark MD

Read this first. Every agent working in this repo follows it.

## What this is

Spark MD, created by the Spark MD team (Ty (Thanh) Doan and Joyce Chen), is an always-on, fully air-gapped lung CT
screening agent that runs on one Dell Pro Precision GB10. It reads incoming chest CTs, tracks nodules across years,
and drafts reports for a radiologist to approve. Patient data never leaves the box. Research demo, not for clinical
use.

This is the Dell Technologies copy of the team's repo. Be concise.

## Read order

1. This file, then `README.md` (features, flow, architecture, how to run).
2. `docs/spark-md-brief.md`: scope, stack, pipeline, risks.
3. `docs/build/README.md` and `docs/build/00-architecture.md`: the build-spec system and the v0 contract (ports, API,
   tools, schema). Then the spec for the layer you're touching (`docs/build/NN-*.md`).
4. `docs/demo-walkthrough.md` (run and reset) and `docs/benchmarks/` (measured results).

## Product rules (apply to any change)

1. **Deterministic code does the medical math.** Measurements, growth, volume doubling time and the Lung-RADS category
   come from code with tests. The LLM orchestrates and writes. It never assigns or changes a Lung-RADS category, and
   never proposes or edits a diameter. Every number in agent-written text must come from a tool result.
2. **Air-gapped at runtime.** No remote LLM or API calls, no CDNs, no web fonts, no telemetry, and no browser Web Speech
   API. Vendor every asset. Set `HF_HUB_OFFLINE=1`, `TRANSFORMERS_OFFLINE=1` and `NEMOCLAW_DISABLE_TELEMETRY=1`. Never
   use `:latest` image tags.
3. **Stack:** OpenShell sandbox + OpenClaw via NemoClaw, Qwen3.8-27B on TensorFold, MONAI lung nodule detection, A7
   rule-based diameter proposals confirmed by the radiologist. Keep the tool layer harness-agnostic (worker REST API +
   MCP).
4. **Human gate.** Only a human click approves a report. The agent has no approve tool, the OpenShell policy does not
   allow the approve route, and the worker rejects approve requests from the sandbox.
5. **Honesty.** Show "Research demo, not for clinical use" on screen. Use real recorded runs and mark sped-up footage.
   Never claim a clinical outcome the data doesn't support. Mark anything not measured as **unverified**.

Unified-memory exhaustion can freeze a GB10. Before any GPU job, run `free -g` and keep `MemAvailable` ≥ 15 GB.

## Ports and addresses (details in `docs/build/00-architecture.md`)

| What | Where |
| --- | --- |
| LLM, OpenAI-compatible | `127.0.0.1:8000`, served name `qwen3.8-27b` (binds `0.0.0.0` so the sandbox reaches `host.openshell.internal:8000`). |
| Imaging worker (API + MCP + UI + SSE) | `10.77.0.1:8100` on dummy interface `spark0` (also `127.0.0.1:8100`). |
| Agent harness (in sandbox) | OpenClaw gateway `:18789`. |
| Sandbox → host | Only through an `enforce`-mode preset with `--trusted-private-host 10.77.0.1`; OpenShell never authorizes loopback egress. |

## Local-only directories (gitignored)

```
data/      DICOM (NLST, LIDC-IDRI), clinical tables, derived labels. Re-downloadable from IDC.
models/    detection, VISTA-3D, ASR and TTS weights. Re-fetchable.
```

Never commit these or container tarballs.

## Working conventions

- Give every measured number its method, its machine, and the date.
- Don't put secrets (HF tokens, hook tokens) in docs. Point to where they live instead.
- Cite data: TCIA NLST CC BY 4.0, LIDC-IDRI CC BY 3.0; IDC as the source.
- Ask before anything hard to reverse: stopping services, deleting data, installing system packages, or anything
  outward-facing.
