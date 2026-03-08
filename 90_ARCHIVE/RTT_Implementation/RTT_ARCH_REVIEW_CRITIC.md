# RTT Implementation Architecture Review (Harsh Critic)

**Project:** RoadSense V2V (RoadSense2 personal docs + `roadsense-v2v` repo)  
**Audience:** Other agents doing architecture/code review  
**Date:** 2026-01-04  
**Scope:** Documentation in `docs/` + current RTT firmware in `roadsense-v2v/hardware/test/espnow_rtt/`

---

## 1) Executive Summary (Read This First)

The low-level RTT mechanics (packet size discipline, buffer/timeout logic, CSV formatting) are *mostly* sound. The system fails at the **operational requirements** level: sessionization (button/scenario logging), non-destructive logging, and distance correlation claims that are not actually supported by the implemented data.

If you proceed to field tests as-is:
- You will get a usable **overall RTT/loss distribution** for broadcast traffic.
- You will **not** be able to produce honest **latency/loss vs distance** plots from V001-only logs.
- You risk corrupting measurements via self-inflicted jitter (SD writes + sensor reads on the timestamping device).

---

## 2) Project Architecture Context (Consistent Across Docs)

The “single-agent / single-logger” architecture is coherent:
- V001 runs the policy (agent) and logs what it **receives** (single-agent perspective).
- V002/V003 are dumb broadcasters.

This matches:
- `CLAUDE.md` (root) “Single-Agent Architecture”
- `docs/00_ARCHITECTURE/ARCHITECTURE_V2_MINIMAL_RL.md` (minimal observation + message age/staleness)
- `docs/90_ARCHIVE/Meeting with professors141225.txt` (one vehicle logs; sessionized scenarios)

Implication: logging formats must reflect what V001 can observe (received peer messages + age), not idealized ground truth.

---

## 3) What’s Actually Implemented (Repo Reality)

### RTT firmware lives in the Git repo
- `roadsense-v2v/hardware/test/espnow_rtt/sender_main.cpp`
- `roadsense-v2v/hardware/test/espnow_rtt/reflector_main.cpp`
- Component headers: `RTTCommon.h`, `RTTPacket.h`, `RTTBuffer.h`, `RTTTimeout.h`, `RTTLogging.h`
- PlatformIO envs: `roadsense-v2v/hardware/platformio.ini` (`env:rtt_sender`, `env:rtt_reflector`)

### Packet schema (hard constraint)
The RTT packet is exactly **90 bytes** (matches `V2VMessage` size), containing:
- `sequence`, `send_time_ms`
- sender GPS (lat/lon/speed/heading)
- sender accel (x/y/z)
- padding

No reflector data is embedded.

### Broadcast behavior
Both sender and reflector use broadcast for send/echo (broadcast→broadcast RTT).

This aligns with BSM-like broadcast *intent*, but changes measurement semantics:
- There is no unicast MAC ACK behavior.
- RTT/2 is an approximation of “one-way age”, not a proof of one-way latency.

---

## 4) Major Gaps / Defects (Blocking Accurate Characterization)

### A) “Distance correlation” is not supported by the logged data
Docs repeatedly imply “log GPS → compute distance between V001 and V002 post-drive”.
That is currently not possible with V001-only RTT logs, because:
- V001 only logs its own GPS snapshot per packet.
- Reflector does not contribute its own lat/lon into the echoed packet.

If you need distance modeling, you must add another information source:
- Reflector periodically broadcasts its GPS (or embeds it into the echoed packet).
- Or you log on both cars and post-merge by time/sequence (harder).

### B) Sessionization (“press button per scenario”) is not implemented
Docs specify button-based start/stop and scenario sessions.
Current RTT sender runs continuously on boot and logs to a single file.

This conflicts directly with the professor-approved workflow:
- press button → record scenario → press button → stop → repeat

### C) Logging is destructive / not field-safe
Current sender opens log file with truncate-on-boot semantics.
That guarantees accidental data loss in real operations.

### D) Measurement contamination risk (sender-induced jitter)
Sender timestamps and also:
- reads IMU,
- parses GPS serial stream,
- scans and writes many SD records, and
- flushes periodically.

Without careful isolation, you risk measuring:
`(network RTT) + (sender scheduling jitter) + (SD blocking time)`.

---

## 5) Random Drive vs Fixed Distance: What You Can Honestly Claim

### If you do a random-distance drive (your stated preference)
**You can produce:**
- overall RTT distribution
- overall packet loss and burstiness statistics
- time series of RTT/loss under “real driving conditions”

**You cannot produce (without extra data):**
- RTT/loss vs distance correlation (because you don’t have both positions)

**Best practice:** accept “no distance model” and compensate with domain randomization ranges.

### If you do fixed-distance tests (10m / 50m / 100m)
This is *much* easier and cleaner:
- You can build a distance-bucketed model without needing dual-GPS merging.
- You can measure distributions per bucket with fewer confounders.
- You can reproduce tests and detect regressions (firmware change, antenna orientation, channel change).

Fixed-distance tests are not “more code”; they are **less code and less analytics**.
They cost more coordination/human procedure, not engineering complexity.

### Hybrid approach (recommended if you want realism + calibration)
1) Fixed-distance benchmarks to get a clean baseline per distance bucket.  
2) One random drive to validate that real-world distributions stay within your domain-randomized ranges.

---

## 6) Recommended Architecture Split (Two Separate Data Products)

Do not conflate these two use-cases:

### Product A: RTT Characterization (for emulator calibration)
Goal: measure network behavior (RTT/loss/jitter) under broadcast payload size.
- Minimal firmware overhead.
- Clear session boundaries.
- Optional: distance buckets via procedure (fixed distances) or via reflector beacons.
- Output: `emulator_params.json` (and possibly distance tiers).

### Product B: Scenario Logs (for training/validation pipeline)
Goal: record what V001 “sees” and what the model should consume.
- Use `DataLogger` single-agent format (`scenario_###.csv` style).
- V001 logs received V002/V003 messages + their age.
- Sessionized by button per scenario (per professor guidance).

Mixing A and B creates misleading requirements and broken expectations in both pipelines.

---

## 7) Concrete, Minimal Fixes (If You Keep RTT Firmware Path)

### Sender firmware hardening
- Implement button-based session start/stop.
- Change file naming to sessionized files (`/rtt_###.csv`) and never truncate prior sessions.
- Make header written once per new file.
- Reduce SD write impact:
  - Write only finalized records (received or timed out).
  - Flush less often (or controlled flush interval), but ensure field safety.

### If you need distance correlation in a random drive
Pick one:
- **Reflector beacons GPS:** reflector periodically broadcasts its own GPS; sender merges by time. (Cleanest single-logger solution.)
- **Echo includes reflector GPS:** reflector modifies payload to include its GPS (but then echo is no longer “same bytes”; you measure a slightly different packet and add reflector compute delay).
- **Dual-logger merge:** log on both cars and merge offline. (Hardest operationally; often fails due to clock drift unless you add sync or sequence coupling.)

---

## 8) Documentation Corrections Needed (Avoid Future Agent Confusion)

These are currently inconsistent or stale and will mislead agents:
- RTT output filename: docs expect `/rtt_data.csv`, implementation uses `/rtt_log.csv`.
- “distance correlation from GPS” claims conflict with V001-only logged fields.
- Some docs mention analysis scripts/paths that don’t exist (`ml/scripts/analyze_espnow_performance.py`).
- There is a mismatch between “IMU/mag optional” vs “IMU required” language across plans.

If you want agents to be productive, the docs must match the repo reality or clearly mark deltas.

---

## 9) Decision Checklist (Answer These Before More Work)

1) **Do you actually need distance correlation** for the emulator MVP, or is “overall distribution + domain randomization” sufficient?
2) **Is broadcast the only intended traffic pattern** (true BSM), or will there be unicast control traffic later?
3) For characterization: do you value **repeatability** (fixed-distance) or **ecological validity** (random drive) more?
4) Are you willing to produce an emulator without distance tiers initially, then add tiers later after adding reflector beacons?

---

## 10) Recommended Next Step (If You Want to Move Fast Without Lying to Yourself)

Option 1 (fastest honest path):
- Run random drive as-is, accept “no distance model”, calibrate emulator to unconditional RTT/loss/jitter, and use wide domain randomization.

Option 2 (best engineering + best science):
- Do fixed-distance characterization first (10m/50m/100m), then one random drive as validation.

Option 3 (random drive + distance tiers):
- Add reflector GPS beacons (single-logger), then do random drive and compute distance tiers.

