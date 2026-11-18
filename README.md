# Spark MD: air-gapped lung CT screening on a Dell Pro Precision GB10

Spark MD is an always-on, fully air-gapped lung cancer screening assistant for a radiology team, running on a single
Dell Pro Precision GB10. It organizes incoming chest CTs, tracks lung nodules across yearly screens, computes the
Lung-RADS v2022 category in deterministic, tested code and drafts a radiology report and a primary-care letter. The
radiologist stays in charge: a person confirms every diameter and approves every report, and patient data never
leaves the box.

**Research demo, not for clinical use.**

## Watch it first

The overview video is attached to the latest release: see [Releases](../../releases).

## How it works

Principle: deterministic code does the medical math; the LLM orchestrates and writes. The agent never proposes or
edits a diameter and never assigns or changes a Lung-RADS category. Every number in agent-written text comes from a
tool result.

| # | Stage | What happens | Check or human gate |
| --- | --- | --- | --- |
| 1 | Intake | A watched folder receives new studies (mock PACS). | Studies stay on the box; no network calls. |
| 2 | Organize (agent) | Parse DICOM headers, group by patient/study/series, pick the thin-slice lung series, write the SQLite registry. | Series selection rules in code (`docs/build/03-intake-registry.md`); every agent tool call is in the on-screen agent log. |
| 3 | Detect (code) | MONAI nodule detection; a rule-based segmenter (A7) proposes a diameter and mask per nodule. | Proposals only: accepted sizes stay empty until a person confirms. |
| 4 | Confirm (human) | The radiologist confirms or edits each diameter in the UI. | **Human gate.** Growth and Lung-RADS are computed only after confirmation. |
| 5 | Score (code) | Lung-RADS v2022 from size, type and growth (growth = >1.5 mm per 12 months; 4X is a category; volume doubling time is evidence only, never a category driver). | Rules engine with test vectors from the ACR table (`tests/rules/`). |
| 6 | Draft (agent) | Radiology report and PCP letter, written from tool results. | Numbers come only from tool results; drafts wait for review. |
| 7 | Approve (human) | Only a person's click approves. | **Human gate.** The agent has no approve tool, the OpenShell policy does not allow the approve route, and the worker rejects approve requests from the sandbox (403). |
| 8 | Deliver | The approved letter lands in a mock EHR inbox. | Nothing is delivered before approval. |

What you see in the UI:

- **Worklist:** patients sorted by highest-priority scan, with the Lung-RADS badge, review state and report status.
- **CT slices:** the study viewer's 2D axial view; detected nodules carry the A7 diameter as a *proposed* measurement
  until the radiologist confirms or edits it.
- **3D anatomy:** lungs, airways, vessels and skeleton meshes with the nodule in focus; the assessment panel shows the
  human-confirmed diameter, growth, VDT and the Lung-RADS category, and the voice/text "Ask the scan" agent.
- **Longitudinal compare:** prior (T0) and current (T1) key slices side by side with the confirmed measurements
  (6.5 mm → 10.4 mm over 357 days in the demo data).

All numbers on screen come from the research demo data; "Research demo, not for clinical use" is shown in the footer.

### Architecture (one box)

| Component | What it does | Where |
| --- | --- | --- |
| LLM | Qwen3.8-27B served by TensorFold, OpenAI-compatible, served name `qwen3.8-27b`, ~36 GiB | `127.0.0.1:8000` (binds `0.0.0.0` so the sandbox reaches it as `host.openshell.internal:8000`) |
| Agent harness | OpenClaw managed by NemoClaw, inside an OpenShell sandbox (restricted tier). Sandbox egress is limited to the imaging worker's agent routes in `enforce` mode; loopback egress is never authorized | in the sandbox; OpenClaw gateway on `127.0.0.1:18789` |
| Imaging worker | MONAI `lung_nodule_ct_detection` v0.5.9 detection, A7 diameter proposals, Lung-RADS v2022 rules engine (tested code), meshes and key slices; REST + MCP + SSE | host, outside the sandbox, on `10.77.0.1:8100` (dummy interface `spark0`; also `127.0.0.1:8100`) |
| UI | Electron + React + three.js served by the worker: worklist, study viewer (2D slices, 3D), agent log, Approve, mock EHR inbox; optional local voice (whisper.cpp ASR, Qwen3-TTS) with chips and mute as the fallback | worker serves `ui/dist` on `:8100` |

The sandbox can reach only the worker's agent routes: not the registry, cache, outbox or the approve route.

## Hardware

**Built and run on**

| Component | Built and run on |
| --- | --- |
| System | Dell Pro Precision GB10 (top configuration) |
| GPU (with memory) | NVIDIA GB10 Grace Blackwell superchip, 128 GB unified LPDDR5x (shared by CPU and GPU) |
| CPU | 20-core NVIDIA Grace Arm CPU (GB10) |
| Memory | 128 GB unified LPDDR5x (shared by CPU and GPU) |
| Storage | 4 TB NVMe SSD |
| OS | NVIDIA DGX OS |
| Software | CUDA 13 (aarch64); Qwen3.8-27B served by TensorFold (OpenAI-compatible); OpenClaw managed by NemoClaw in an OpenShell sandbox; MONAI lung_nodule_ct_detection v0.5.9; SQLite; Electron + React + three.js UI; optional whisper.cpp ASR and Qwen3-TTS |

**Also meets the GPU requirement (not tested on these)**

The requirement is 128 GB of GPU-addressable memory, NVIDIA, Linux. Spark MD was not run on this system.

| System | Note |
| --- | --- |
| Dell Pro Precision GB300 | Meets the GPU requirement; not tested |

## What's in this repo

| Path | What |
| --- | --- |
| `sparkmd/` | Imaging worker (FastAPI): intake, registry, imaging pipeline, Lung-RADS rules, REST + MCP + SSE, demo harness |
| `ui/` | Electron + React + three.js desktop app, offline (see `ui/README.md`) |
| `agent/` | OpenShell policy preset, MCP wiring, tools profile, agent skill and hooks for OpenClaw (see `agent/README.md`) |
| `bin/` | `sparkmd-start`, `sparkmd-stop`, `sparkmd-status`, `sparkmd-sudo`, `sparkmd-demo`, `sparkmd-baseline` and voice helpers |
| `scripts/` | Dev and fixture helpers, backlog seeding, the optional offline Sybil precompute script |
| `fixtures/` | Demo fixtures and seeds (`sybil_risk.json` is shipped empty, see Known limits) |
| `tests/` | Rules, imaging, worker, voice, ask and acceptance tests |
| `docs/spark-md-brief.md` | The brief: scope, stack, pipeline, risks |
| `docs/build/` | Per-layer build specs: architecture contract, setup, LLM serving, intake, imaging, rules, worker API, agent, UI, voice, demo harness, acceptance |
| `docs/benchmarks/` | Measured results with method and raw data |
| `docs/demo-walkthrough.md` | Run-and-reset runbook for the demo |
| `docs/deck-assets/` | Key CT slices rendered by the imaging pipeline |
| `assets/` | Screenshots of the UI |
| `video/` | HyperFrames project used for the overview video |

## Setup (Linux, NVIDIA DGX OS on the GB10)

These steps are the repo's own; per-layer commands and gotchas are in `docs/build/`.

1. Start from a Dell Pro Precision GB10 on NVIDIA DGX OS with Docker usable by your user and GPU access in containers
   (`docs/build/01-box-setup.md`). Keep at least 15 GiB `MemAvailable` (`free -g`) before any GPU job.
2. Stage what is not in this repo (all required, voice optional): the Qwen3.8-27B weights (~36 GiB) and the TensorFold
   serving directory (default `~/qwen38-27b-tensorfold`, see `docs/build/02-llm-serving.md`); `models/` with the MONAI
   `lung_nodule_ct_detection` v0.5.9 bundle, the VISTA-3D lung/lobe model and, for voice, the whisper.cpp and Qwen3-TTS
   models; `data/` with the NLST and LIDC-IDRI DICOM studies and clinical tables (from NCI Imaging Data Commons); and
   the offline package caches `wheelhouse/` (Python) and `nodehouse/` (npm).
3. Create the Python 3.12 venv offline: `python3.12 -m venv ~/.venvs/spark-md` and
   `~/.venvs/spark-md/bin/python -m pip install --no-index --find-links <wheelhouse>/wheels -r <wheelhouse>/requirements.lock`.
4. Build the UI offline from `nodehouse/` as in `ui/README.md` (`npm ci`, then `npx tsc -b && npx vite build`; the worker
   serves `ui/dist`).
5. Install NemoClaw, onboard the OpenShell sandbox `spark-md-openclaw` (provider `vllm`, `restricted` tier, telemetry
   off) and run the `agent/setup/` scripts in order (`docs/build/07-agent.md`, `agent/README.md`).
6. Write a `.env` with the agent hook token (`agent/setup/50-hooks.sh` generates it), and set `SPARKMD_HOME`,
   `SPARKMD_ENV_FILE`, `SPARKMD_PYTHON` and `SPARKMD_MODELS` if your paths differ from the defaults in `bin/sparkmd-lib.sh`.
7. Once, at the console: `sudo bash bin/sparkmd-sudo` (creates `spark0`, blocks ports 8000 and 8100 on physical
   interfaces, fixes the Docker socket ACL and the Electron sandbox).
8. Start everything: `bin/sparkmd-start` (preflight → spark0 → firewall → LLM → sandbox → worker → voice → UI). Check with
   `bin/sparkmd-status` (exit code = number of FAIL lines); stop with `bin/sparkmd-stop`. Both `start` and `stop` take
   `--only llm,sandbox,worker,asr,tts,ui` and `--dry-run`. The scripts set `HF_HUB_OFFLINE=1`, `TRANSFORMERS_OFFLINE=1`
   and `NEMOCLAW_DISABLE_TELEMETRY=1` and never use `:latest` image tags.
9. Run the demo: `bin/sparkmd-demo` (reset, launch the app, replay the demo patient's screens).
10. The full run-and-reset checklist is `docs/demo-walkthrough.md`; the end-to-end acceptance checks are in
    `docs/build/11-acceptance.md`.

## Known limits

- **Not in this repo:** the Qwen3.8-27B weights (~36 GiB) and TensorFold, the MONAI lung_nodule_ct_detection v0.5.9
  bundle, the optional whisper.cpp and Qwen3-TTS models, the NLST/LIDC-IDRI DICOM studies and the offline package
  caches. You stage them yourself (Setup, step 2).
- **Dell has not re-run this copy.** It is the Spark MD team's work as they built and ran it.
- **The LLM binds `0.0.0.0:8000`** so the sandbox can reach it. `bin/sparkmd-sudo` blocks the port on physical
  interfaces; check this on your network before you run it.
- **No run time is stated** for the pipeline or the demo.
- **Diameters are radiologist-confirmed.** A7 output is a proposal, not an accepted measurement. In the team's
  held-out checks (`docs/benchmarks/`), ±3 mm detector-box shifts moved A7 by up to 1.8 mm, which is why a person
  confirms each diameter.
- **VISTA-3D nodule segmentation failed the team's validation** and is not used for any size, growth, VDT or
  Lung-RADS input; VISTA-3D is used only for the lung/lobe mask, with a CPU threshold fallback.
- **Growth normalized to 12 months is the team's interpretation.** Lung-RADS Note 6 defines growth as >1.5 mm "within
  a 12-month interval". For longer intervals Spark MD rate-normalizes the change to a per-12-month value (short
  intervals are never scaled up). This is not ACR wording, and the UI rule text labels it.
- **The Sybil research risk card is left out of this copy.** `fixtures/sybil_risk.json` ships with no scores, so the
  card shows nothing, until the licence of the Sybil checkpoints is confirmed. If you install Sybil (a separate
  open-source project) yourself, `scripts/sybil_precompute.py` regenerates the fixture. The card is never validation:
  Sybil was trained on NLST and the demo patients may be in its training set; it never feeds Lung-RADS and the agent
  does not see it.
- **Unverified:** real microphone capture and speaker playback; OpenShell policy persistence across reboot; the full
  stack (LLM + agent + imaging + voice) loaded together under load; the complete acceptance pass in
  `docs/build/11-acceptance.md`.

## Credits

- **Created by the Spark MD team:** Ty (Thanh) Doan and Joyce Chen. Original repo:
  [tdoan35/spark-md](https://github.com/tdoan35/spark-md). Please star it.
- **Partner:** NVIDIA (GB10 Grace Blackwell superchip, DGX OS, CUDA, NemoClaw, OpenShell).
- **Software:** Qwen3.8-27B, TensorFold, OpenClaw, NemoClaw, OpenShell, MONAI and the `lung_nodule_ct_detection`
  bundle, VISTA-3D (`nvidia/NV-Segment-CT`), Sybil, whisper.cpp and Qwen3-TTS, each under its own licence (see
  `THIRD_PARTY_NOTICES.md`). No third-party repository is bundled in this copy; nothing in the setup clones one.
- **Data:** National Lung Screening Trial (NLST) via The Cancer Imaging Archive (TCIA), CC BY 4.0,
  doi:10.7937/TCIA.HMQ8-J677; LIDC-IDRI via TCIA, CC BY 3.0, doi:10.7937/K9/TCIA.2015.LO9QL9SX (Armato et al. 2011);
  accessed through NCI Imaging Data Commons (Fedorov et al. 2023). Lung-RADS v2022 is from the American College of
  Radiology (ACR).
- **AI-generated content:** spoken answers use a synthetic voice (Qwen3-TTS, generated on the device and labelled on
  screen). Much of the code was written by AI coding agents under the team's supervision, from the specs in
  `docs/build/`.
- **Trademarks:** Dell, Dell Technologies and Dell Pro Precision are trademarks of Dell Inc. or its subsidiaries.
  NVIDIA, Grace, Blackwell, CUDA and DGX are trademarks of NVIDIA Corporation. Other names are trademarks of their
  respective owners. Naming a third-party product or project does not imply that its owner endorses this demo or Dell.

## Support

Logan Lawler, Dell Technologies: [logan.lawler@dell.com](mailto:logan.lawler@dell.com)

## License

The Spark MD team's repository has no licence file, so their code is not relicensed here (it is not MIT): the authors
keep all rights, and Dell shares it with their permission to download, run and evaluate as a demo. Dell's changes carry
the same terms. See [`LICENSE`](LICENSE) and [`THIRD_PARTY_NOTICES.md`](THIRD_PARTY_NOTICES.md).
Copyright (c) 2026 Dell Inc. (changes by Dell Technologies).
