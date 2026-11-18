# Third-party notices

## Upstream repository

`tdoan35/spark-md` (the Spark MD team: Ty (Thanh) Doan and Joyce Chen). The upstream repository has no licence file
and no notice file; there is no upstream notice text to reproduce. See `LICENSE`. No source file in this repo carries
its own licence header, and none has been removed.

## Bundled third-party code

None. No third-party repository is copied into this repo.

## Components you install separately (not in this repo)

Each is used under its own licence, which you accept when you obtain it. Licence details are given here only where
the repo states them.

| Component | Used for | Licence as stated in the repo |
| --- | --- | --- |
| Qwen3.8-27B weights | LLM | See the model's own licence |
| TensorFold (`tensorfold-qwen38-27b:v0.6.0` image) | LLM serving | See the project's own licence |
| OpenClaw, NemoClaw, OpenShell | Agent harness and sandbox | See each project's own licence |
| MONAI and the `lung_nodule_ct_detection` bundle v0.5.9 | Nodule detection | See the project's and bundle's own licences |
| VISTA-3D (`nvidia/NV-Segment-CT`, `model.pt`) | Lung/lobe mask | NVIDIA Open Model License |
| Sybil | Optional offline precompute of the research risk card (`scripts/sybil_precompute.py`); no Sybil scores ship in this repo | See the project's own licence |
| whisper.cpp and its models | Optional speech recognition | See the project's own licence |
| Qwen3-TTS 1.7B-Base | Optional spoken answers (synthetic voice) | See the model's own licence |
| Community faster-qwen3-tts container image by mARTin-B78 (`v6.15.0`, pinned by digest) | Optional TTS serving | MIT (community build) |
| Python packages (`wheelhouse/requirements.lock`, `pyproject.toml`) | Worker | Each package's own licence |
| npm packages (`ui/package.json`, `ui/package-lock.json`, `video/package.json`) | UI and video project | Each package's own licence |

## Data

- National Lung Screening Trial (NLST), The Cancer Imaging Archive (TCIA), CC BY 4.0, doi:10.7937/TCIA.HMQ8-J677.
- LIDC-IDRI, The Cancer Imaging Archive (TCIA), CC BY 3.0, doi:10.7937/K9/TCIA.2015.LO9QL9SX (Armato et al. 2011).
- Accessed through NCI Imaging Data Commons (Fedorov et al. 2023).
- Lung-RADS v2022, American College of Radiology (ACR).

Screenshots in `assets/` and the CT slices in `docs/deck-assets/` show images rendered from this CT data (credits
above).
