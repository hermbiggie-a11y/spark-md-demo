# Spark MD: run-and-reset runbook

Research demo, not for clinical use. A person does every Confirm and Approve; the replay never does.

## 0. Before you start

1. `bin/sparkmd-status`: LLM, worker, sandbox/gateway, whisper, TTS, firewall and duplicates all OK. If not, run
   `bin/sparkmd-start` (and `sudo bash bin/sparkmd-sudo` at the console if it prints sudo hints).
2. Don't run test or QA scripts while presenting: they write rows into the on-screen agent log.

## 1. Launch

1. On the box: `bin/sparkmd-demo --wait` resets the demo (re-seeds the cohort, zeroes the network counters), launches
   the app and waits for Enter. `DEMO_BASELINE=1 bin/sparkmd-demo --wait` restores a saved human-reviewed baseline
   instead (`bin/sparkmd-baseline save` creates it).
2. Maximize the app. Voice: mute (speaker icon) unless the microphone was tested.
3. Optional, to show the air gap: unplug the LAN cable and turn Wi-Fi off. The System page's network monitor should
   read ~0 bytes sent with links down.

## 2. A new scan arrives (press Enter)

- The "Time machine replay" banner shows the hero patient's T0 screen; the patient appears in the worklist.
- Agent log (keep it visible): get_study → select_series → run_analysis.
- Wait for the banner "Waiting for radiologist".

## 3. Radiologist review of T0

1. Open the hero patient's study.
2. **3D anatomy:** toggle Vessels/Skeleton once; "Focus nodule".
3. **CT slices:** click a nodule marker above the slider; the nodule is boxed.
4. **Assessment:** the nodules are marked Proposed (LLL ~6.5 mm, RUL ~5.0 mm) → **Confirm all** (or edit a diameter
   first).
5. **Lung-RADS 3**, "6-month LDCT" appears, computed by code, not the AI.
6. Agent log: "Confirmed: Lung-RADS 3. Drafting reports." → wait for the drafts.
7. **Review & approve** → radiology report → **Approve** → PCP letter → **Approve** → toast "Delivered to Primary care
   physician".

## 4. One year later: T1 arrives by itself

1. The banner switches to T1 after the T0 approval; the agent triages it. Wait for "Waiting for radiologist".
2. Open T1 (pill **T1**; T0 ✓). **Compare** tab: prior 6.5 mm → current 10.4 mm, +3.9 mm over 357 days; both key slices
   boxed.
3. **Confirm** → **Lung-RADS 4B** + management (diagnostic CT / PET-CT / tissue sampling or referral).
4. The agent sets priority **elevated**: the patient moves to the top of the worklist and appears under **Priority
   patients**.

## 5. Talk to the scan (after the T1 drafts)

- Chips, in order: **"Show me the nodule"** (the camera flies to it) → **"Compare it to last year"** → **"Why did you
  flag this?"**. Every number matches the Assessment panel.
- With voice (only if tested): hold Space and say the exact chip phrase.

## 6. Approve T1 and close

1. **Review & approve** → radiology → **Approve** → PCP letter → **Approve**.
2. **Inbox:** the letters for T0 and T1.
3. Close on the network monitor and the footer ("Research demo, not for clinical use · Local only · Air-gapped").

## Reset and retake

Re-run `bin/sparkmd-demo --wait` (or `DEMO_BASELINE=1 bin/sparkmd-demo --wait`), then press Enter.

## Don't

- Open 100288 (failed proposal) or 100285 (false-positive 27 mm candidate) while presenting.
- Edit a diameter unless it's a planned beat.
- Show sped-up footage without an on-screen "2×" or "time skipped" label.

## Fallbacks

- **LLM:** restart → lower memory → other checkpoint → no-LLM template mode (`docs/build/02-llm-serving.md`).
- **Imaging speed:** live → cached results, labelled as such (`docs/build/10-demo-harness.md`).
- **UI on the box:** Electron with the default ANGLE backend + `--ozone-platform=x11`; last resort the same app in a
  browser at `http://127.0.0.1:8100/`.
- **Stop everything:** `bin/sparkmd-stop` (leaves `spark0`, the registry, `cache/` and `outbox/` alone).
