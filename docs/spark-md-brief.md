# Spark MD — Project Brief

Research demo, not for clinical use.

## One-liner

Spark MD is an always-on, fully air-gapped lung cancer screening assistant for hospital radiology teams. It reads
incoming chest CTs, finds and tracks nodules across years, and drafts reports for a radiologist to approve. It runs on
one Dell Pro Precision GB10, and patient data never leaves the box.

## Team

Created by the Spark MD team: **Ty (Thanh) Doan** (engineering: infra, imaging worker, agent tools, UI) and
**Joyce Chen** (story, demo video and QA).

## Why local

Patient scans can't leave the hospital. Spark MD acts on triggers (new scans arriving), not chat prompts, and the
OpenShell sandbox has a real job: the agent can reach only the imaging worker's agent routes, so it cannot exfiltrate
data or approve its own work.

## Stack

- **Sandbox:** OpenShell.
- **Agent harness:** OpenClaw via NemoClaw inside OpenShell. See `docs/build/07-agent.md`.
- **LLM:** Qwen3.8-27B on TensorFold, `tensorfold-qwen38-27b:v0.6.0`,
  `HOST=0.0.0.0 PORT=8000 SERVED_NAME=qwen3.8-27b PARALLEL=4 CONTEXT=65536 KV_POOL_GB=16` (binds `0.0.0.0` so the
  sandbox bridge reaches it at `host.openshell.internal:8000`), ~36 GiB footprint. No prefix caching, so keep agent
  sessions warm. See `docs/build/02-llm-serving.md`.
- **Imaging:** MONAI `lung_nodule_ct_detection` v0.5.9 (on monai 1.6.1 with documented shims,
  `docs/build/04-imaging-pipeline.md`), A7 rule-based diameter proposals confirmed by the radiologist, VISTA-3D
  (`nvidia/NV-Segment-CT`) for the lung/lobe mask only with a threshold fallback, Lung-RADS v2022 rules engine with test
  vectors (`docs/build/05-clinical-rules.md`).
- **Imaging validation:** VISTA-3D nodule segmentation failed the team's LIDC boundary and crop-stability evaluation,
  so no automatic nodule measurement is accepted: a person confirms every diameter. See
  `docs/benchmarks/2026-10-24-vista-longitudinal.md` and `docs/benchmarks/2026-10-24-nodule-holdout.md`.
- **Data:** NLST (TCIA, CC BY 4.0) for multi-year patients; LIDC-IDRI (TCIA, CC BY 3.0) for radiologist-annotated
  ground truth. Staged in `data/` (not in this repo).
- **Staged weights (`models/`, not in this repo):** VISTA-3D `nvidia/NV-Segment-CT` `model.pt` (NVIDIA Open Model
  License) and the MONAI `lung_nodule_ct_detection` bundle v0.5.9.

## Architecture

**Principle:** deterministic code does the medical math; the LLM orchestrates and writes. The LLM never assigns the
Lung-RADS category.

- **Agent** (in the OpenShell sandbox) calls the imaging worker's tools over a stable host IP (`10.77.0.1`, a dummy
  interface that survives a cable pull). OpenShell never authorizes loopback egress (see
  `docs/build/00-architecture.md`).
- **Imaging worker** (host GPU service, outside the sandbox): MONAI detector, A7 proposals, Lung-RADS rules,
  marching-cubes meshes.
- **LLM server** on localhost `:8000` serving Qwen3.8-27B.
- **UI:** worklist, three.js 3D viewer, voice commands, Approve button, mock EHR inbox.
- **Egress policy:** the sandbox may reach only the imaging worker's agent routes, in `enforce` mode (the default is
  audit, which allows). Nothing else. The approve route is human-only.

### Pipeline

1. **Intake:** a watched folder acts as a mock PACS.
2. **Organize** *(agent):* parse DICOM headers, group by patient/study/series, pick the thin-slice lung series, write to
   a SQLite registry.
3. **Preprocess:** HU conversion and resampling with the MONAI bundle's transforms.
4. **Lung mask:** VISTA-3D lobes (CPU threshold fallback).
5. **Detect:** MONAI candidates; drop any outside the lung mask.
6. **Propose:** A7 diameter and mask per nodule, as a proposal.
7. **Confirm** *(human):* the radiologist confirms or edits each diameter.
8. **Compare with priors:** match nodules to last year's scan by lobe and position; compute growth from confirmed
   diameters.
9. **Score** *(rules):* Lung-RADS from size, type, growth.
10. **Render:** meshes to three.js, plus key 2D slices with the nodule boxed.
11. **Report** *(agent):* radiology-style report and PCP letter.
12. **Human gate:** the radiologist approves; the letter goes to the mock EHR inbox.

### Voice (optional)

Local push-to-talk recognition uses whisper.cpp small.en on CUDA with phrase prompting + Silero VAD. Four fixed
commands: show nodule, compare last year, explain the flag, reset view. Viewer actions run immediately in
deterministic code, are logged, and the agent explanation follows. Spoken answers use local Qwen3-TTS 1.7B-Base with
a synthetic voice, after the answer finishes and the shared LLM is idle; the voice is labelled on screen. Real
microphone capture and playback remain **unverified**. See `docs/build/09-voice.md`.

## Risks

| Risk | Test / mitigation |
| --- | --- |
| **Memory contention** | LLM, agent, imaging and voice share 128 GB of unified memory. Keep `MemAvailable` ≥ 15 GiB before any GPU job; run one heavy GPU job at a time. The full stack under load together is **unverified**. |
| **Arm64** | MONAI, SimpleITK and pydicom must install and run on DGX OS (aarch64); the offline wheelhouse pins the versions. |
| **Sandbox → host reach** | The sandbox reaches the worker on `10.77.0.1` (loopback egress is never allowed by OpenShell), with `enforce` denials visible. |
| **Measurement reliability** | A7 is a proposal; the radiologist confirms or edits every diameter before growth or Lung-RADS are computed. |
| **Data licensing** | NLST (CC BY 4.0) and LIDC-IDRI (CC BY 3.0) both allow use with attribution. Credit TCIA wherever images are shown. |

## Honesty rules

- "Research demo, not for clinical use" on screen.
- Real recorded runs. Any sped-up footage gets a visible "2×".
- The agent log stays on screen during the time-machine replay.
- Never claim an outcome the data doesn't support.
