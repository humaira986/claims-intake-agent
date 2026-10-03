# System 4 — Orchestration Evidence

## 1. SQL-filtered defect slice

Command:

python -c 'from pathlib import Path; from shift_monitor.warm import WarmStore; db=WarmStore(Path("data/warm.sqlite")); rows=db.defects_since("2026-04-26T00:00:00Z", 50); print("SQL-filtered defects:", len(rows)); [print(r) for r in rows]; print("QUERY PLAN:"); print(*db.explain_defects_since("2026-04-26T00:00:00Z", 50), sep="\n")'

Observed:

SQL-filtered defects: 2

{'id': 'DEF-20260429-001', 'ts': '2026-04-29T12:05:28Z', 'shift': 'A', 'component': 'coolant-loop-CL-5', 'severity': 'medium', 'description': 'Pinhole leak identified at elbow joint upstream of pump P-2, drip rate approximately 1 drop per minute. Temporary clamp applied, weld repair scheduled.'}

{'id': 'DEF-20260427-001', 'ts': '2026-04-27T01:19:46Z', 'shift': 'C', 'component': 'capacitor-bank-C-7', 'severity': 'critical', 'description': 'Internal short circuit detected during final QA; impedance dropped to 0.08 ohm from expected 4.5 ohm. Unit segregated for destructive analysis.'}

QUERY PLAN:

(7, 0, 0, 'SEARCH defects USING INDEX idx_defects_ts (ts>?)')

The defect slice is filtered in SQL using the timestamp predicate. The query plan confirms index usage.

## 2. Hot state size

Command:

wc -c data/hot_state.json

Observed:

680 data/hot_state.json

The hot state is 680 bytes, which is below the 5 KB target.

## 3. Recovery behavior

Observed:

recent 10 min: resume
stale 31 min: fresh
empty: fresh
complete: fresh
threshold: 30 minutes

The recovery logic resumes an incomplete state when the latest step is within 30 minutes. A stale state, empty state, or already-complete state starts fresh.

## 4. Fork isolation

Observed:

base unchanged: True

fork A hot state: True

fork B hot state: True

fork A scratchpad:
{"hypothesis_id":"hypothesis-A","evidence":"finding-A","conclusion":"Hypothesis A isolated","ts":"2026-10-01T10:41:53.260719Z"}

fork B scratchpad:
{"hypothesis_id":"hypothesis-B","evidence":"finding-B","conclusion":"Hypothesis B isolated","ts":"2026-10-01T10:41:53.260719Z"}

forks isolated: True

The base hot state remains unchanged. Each hypothesis receives its own hot-state copy and its own scratchpad, demonstrating fork isolation.

## 5. Shift C execution

Command:

shift-monitor run-shift --shift C --warm-db data/warm.sqlite --since 2026-04-26T00:00:00Z --recorded-response ./fixtures/recorded_responses/shift_C_2026-04-30.json

Observed:

shift C: 2 new defects

Shift C 2026-04-30: 3 high + 2 medium defects on capacitor-bank-C-7, all from lot 2026-0430-B (DAR 0.39, ESR ~19 mOhm); 1 low VP-4 vent squeal (repeat). Lot quarantine recommended.

## 6. Test verification

33 passed in 0.53s