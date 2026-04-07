# Forge Timing Profiles

Abstracted speed levels for network-active campaigns. Campaigns reference these profiles instead of hardcoding rate limits. The operator selects a profile based on their network environment; the campaign applies the corresponding parameters.

---

## Profiles

| Profile | Max PPS | Threads | Timeout (s) | Retries | Use Case |
|---------|---------|---------|-------------|---------|----------|
| **T1 — Cautious** | 40 | 25 | 10 | 2 | Residential networks, shared infrastructure, unknown constraints |
| **T2 — Moderate** | 150 | 50 | 5 | 1 | Dedicated connection, known network capacity, no upstream rate limiting |
| **T3 — Aggressive** | 500 | 100 | 3 | 0 | Cloud/VPS, lab environments, no upstream constraints |

**Default: T1.** When the operator's network environment is unknown, campaigns use T1. This avoids triggering residential router rate limits or upstream throttling.

---

## Parameter Definitions

| Parameter | Description |
|-----------|-------------|
| **Max PPS** | Maximum packets per second. The campaign's primary rate-limiting knob. Tools like httpx use `-rl`, nmap uses `--max-rate`. |
| **Threads** | Maximum concurrent connections/threads. Controls parallelism independent of packet rate. |
| **Timeout** | Per-connection timeout in seconds. Lower timeouts suit fast networks; higher timeouts accommodate latency. |
| **Retries** | Number of retry attempts on timeout/failure. Cautious profiles retry to avoid false negatives; aggressive profiles skip retries for speed. |

---

## Pipeline Guidance

### Strategist

When writing a CONOPS for a campaign that involves network activity, include timing profile recommendations in the Practitioner Context section. Default to T1 unless the raw ask or environment investigation suggests otherwise. Write the specific parameter values from the selected profile into the CONOPS — downstream stages read the CONOPS, not this reference file.

### Planner

The planner does not read this file. Timing information reaches the planner through the CONOPS. When the CONOPS Practitioner Context includes timing profiles, carry those values into the plan's Infrastructure Requirements and methodology sections.

### Assembler

The assembler does not read this file. Timing information reaches the assembler through the plan. When generating config defaults for network-active artifacts, include `timing_profile`, `rate_limit`, and `threads` fields using values from the plan. Generated skill instructions should note that the operator can adjust the timing profile in config.

---

## Design Rationale

Timing profiles are constants, not operator interview questions. The profiles exist as a shared vocabulary across the pipeline — the strategist reads this reference and writes specific values into the CONOPS. Everything downstream reads the CONOPS and plan, not this file.

The operator's specific network constraints (router firmware limits, ISP throttling, VPN overhead) determine which profile they select. The campaign does not need to know why T1 was chosen — only that T1 means 40 pps / 25 threads.
