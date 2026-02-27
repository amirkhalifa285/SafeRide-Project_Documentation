# CONVOY FIELD VALIDATOR Agent Prompt

**Purpose:** Run convoy post-drive validation immediately on site and give a clear GO/NO-GO verdict before leaving.

**Last Updated:** February 27, 2026

---

## Context

RoadSense recording protocol is now **mesh-aware ego logging**:
- `V001` (ego) logs `V001_tx_*.csv` and `V001_rx_*.csv`
- `V002/V003` run powered mesh-relay firmware (no SD required)

The analyzer script now supports both input modes:
1. `ego_only` mode (default now): only `V001_tx` + `V001_rx`
2. `full` mode (legacy): `V001/V002/V003` each with `tx+rx`

Your job is to run the script, inspect outputs, and tell Amir if recording quality is acceptable.

---

## Step 1: Run Analyzer On Site

```bash
cd /home/amirkhalifa/RoadSense2/roadsense-v2v/ml
source venv/bin/activate
python scripts/analyze_convoy_recording.py \
  --input-root <RECORDING_DIR> \
  --out-dir /home/amirkhalifa/RoadSense2/roadsense-v2v/ml/data/convoy_analysis_site \
  --no-plots
```

Where `<RECORDING_DIR>` is where the CSV files were copied after drive.

### Accepted input layouts

#### Ego-only (recommended/current)
Either of these is accepted:
```text
<RECORDING_DIR>/
├── V001_tx_<NNN>.csv
└── V001_rx_<NNN>.csv
```
or
```text
<RECORDING_DIR>/V001/
├── V001_tx_<NNN>.csv
└── V001_rx_<NNN>.csv
```

#### Full (legacy)
```text
<RECORDING_DIR>/V001/V001_tx_<NNN>.csv
<RECORDING_DIR>/V001/V001_rx_<NNN>.csv
<RECORDING_DIR>/V002/V002_tx_<NNN>.csv
<RECORDING_DIR>/V002/V002_rx_<NNN>.csv
<RECORDING_DIR>/V003/V003_tx_<NNN>.csv
<RECORDING_DIR>/V003/V003_rx_<NNN>.csv
```

### Expected outputs

- `data_quality_summary.json`
- `analysis_summary.json`
- `convoy_events.json`
- `convoy_emulator_params.json`
- `convoy_analysis_report.md`
- `convoy_observations.npz`

---

## Step 2: Determine Mode

Read `mode` from outputs:
- `data_quality_summary.json["mode"]`
- `analysis_summary.json["mode"]`

It will be either:
- `"ego_only"`
- `"full"`

Use the matching verdict rules below.

---

## Step 3A: GO/NO-GO Rules For `ego_only` Mode

### 1) File integrity (must pass)
From `data_quality_summary.json -> files`:
- `V001_tx` and `V001_rx` both exist
- `rows_valid >= 500` on both
- `malformed_rows <= 5%` of total
- `gaps_gt_500ms <= 10`
- `sensor_status != "FAIL"`

### 2) GPS and movement sanity (must pass)
- `V001_tx.gps_fix_pct >= 95%`
- V001 speed is non-zero during drive segment (not parked-only recording)

### 3) Mesh reception health (must pass)
From `analysis_summary.json -> pdr_by_link`:
- at least 2 sender links should exist (typically `V002->V001` and `V003->V001`)
- each link estimated `pdr >= 0.80`
- no link below `0.70`

Notes:
- In `ego_only` mode, link metrics are **estimated from RX message spacing** (not exact cross-device PDR).

### 4) Relay evidence (must pass)
From `V001_rx_*.csv`:
- column `hop_count` exists
- at least some rows with `hop_count >= 1`

### 5) Hard braking quality (must pass)
From `convoy_events.json -> detected.hard_event`:
- minimum `min_accel_x_ms2 < -2.0` (minimum acceptable)
- target quality is `< -3.0`

### 6) Stationary tail (warning-level)
From `convoy_events.json -> detected.stationary_tail`:
- `>= 45s` preferred
- `20-45s` warning, still usable

---

## Step 3B: GO/NO-GO Rules For `full` Mode (legacy)

Use stricter classical checks:
- all 6 files valid
- all 6 directional links PDR >= 0.80
- no link below 0.70
- hard braking `min_accel_x_ms2 < -2.0` (target `< -3.0`)

---

## Step 4: Quick Verdict Command

Run this helper after analyzer finishes (works for both modes):

```bash
python - <<'PY'
import csv, json
from pathlib import Path

out = Path('/home/amirkhalifa/RoadSense2/roadsense-v2v/ml/data/convoy_analysis_site')
dq = json.loads((out/'data_quality_summary.json').read_text())
an = json.loads((out/'analysis_summary.json').read_text())
ev = json.loads((out/'convoy_events.json').read_text())

mode = an.get('mode') or dq.get('mode')
files = dq.get('files', [])

def find_file(t):
    for f in files:
        if f.get('type') == t:
            return f
    return {}

tx = find_file('tx')
rx = find_file('rx')

hard = (ev.get('detected') or {}).get('hard_event') or {}
hard_min = hard.get('min_accel_x_ms2', 999)

pdr_links = an.get('pdr_by_link', {})
pdr_vals = [v.get('pdr', float('nan')) for v in pdr_links.values() if isinstance(v, dict) and 'pdr' in v]

file_ok = bool(tx and rx and tx.get('rows_valid',0) >= 500 and rx.get('rows_valid',0) >= 500)
gps_ok = tx.get('gps_fix_pct', 0) >= 95
pdr_ok = len(pdr_vals) >= 2 and all((p == p) and p >= 0.80 for p in pdr_vals) and all(p >= 0.70 for p in pdr_vals)
hard_ok = hard_min < -2.0

# hop evidence from rx CSV
hop_ok = False
rx_file = Path(rx.get('file', ''))
if rx_file.exists():
    with rx_file.open() as f:
        r = csv.DictReader(f)
        if 'hop_count' in (r.fieldnames or []):
            for row in r:
                try:
                    if int(float(row['hop_count'])) >= 1:
                        hop_ok = True
                        break
                except Exception:
                    pass

print('mode=', mode)
print('file_ok=', file_ok, 'gps_ok=', gps_ok, 'pdr_ok=', pdr_ok, 'hop_ok=', hop_ok, 'hard_ok=', hard_ok, 'hard_min=', hard_min)
print('VERDICT=', 'GO' if all([file_ok, gps_ok, pdr_ok, hard_ok, hop_ok]) else 'NO_GO')
PY
```

---

## Step 5: Output Format To Amir

Use this concise format:

```text
CONVOY VALIDATION — <GO / NO-GO>
Mode: <ego_only/full>

Checks:
- File integrity: PASS/FAIL
- GPS fix on V001 TX: PASS/FAIL
- Link health (PDR): PASS/FAIL
- Relay evidence (hop_count>=1): PASS/FAIL
- Hard braking quality: PASS/FAIL (min_accel_x_ms2=...)

Decision:
- GO: recording accepted
or
- NO-GO: re-record now; include exact fix (board placement, braking strength, GPS lock timing)
```

---

## Operational Notes

- If analyzer fails to find files, verify naming and directory placement first.
- Always pass `--input-root`; do not rely on script defaults.
- In `ego_only` mode, estimated PDR is sufficient for on-site acceptance decisions.
- Final architecture still uses V001 as ego for production and recording.
