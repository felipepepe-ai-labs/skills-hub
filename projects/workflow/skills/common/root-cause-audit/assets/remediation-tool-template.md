# Remediation Tool Template

Generic design for repairing records affected by confirmed causes. Adapt names and technology (PL/SQL, SQL script, Python, Node). Only build it if the user asks.

## Principles

1. Three modes, in order of risk: `CHECK` (list), `SIMULATE` (show what would change, write nothing to business tables), `EXECUTE` (apply).
2. `EXECUTE` refuses to run unless a `SIMULATE` of the same scope exists in `logs` and nothing changed since.
3. Every affected entity is attributed to **one** cause (fixed priority) and gets a proposed action.
4. Causes pending a business decision are reported as `PENDING_DECISION` and are **never** executed.
5. Everything is recorded: `logs` (one row per run) and `trazas` (one row per entity).
6. Idempotent: running `EXECUTE` twice does not duplicate or corrupt.

## Tables

```sql
-- One row per run
CREATE TABLE rca_logs (
  run_id        NUMBER PRIMARY KEY,
  mode          VARCHAR2(10) NOT NULL,      -- CHECK | SIMULATE | EXECUTE
  started_at    TIMESTAMP NOT NULL,
  finished_at   TIMESTAMP,
  scope         VARCHAR2(400),              -- parameters used
  total_checked NUMBER, total_affected NUMBER,
  total_fixed   NUMBER, total_pending NUMBER, total_errors NUMBER,
  status        VARCHAR2(20)                -- OK | ABORTED | ERROR
);

-- One row per entity
CREATE TABLE rca_trazas (
  run_id      NUMBER NOT NULL REFERENCES rca_logs(run_id),
  entity_id   VARCHAR2(100) NOT NULL,
  cause_id    VARCHAR2(20),                 -- H1.. as confirmed in the audit
  state       VARCHAR2(20) NOT NULL,        -- DETECTED | WOULD_FIX | FIXED | PENDING_DECISION | ERROR
  proposed    VARCHAR2(400),                -- proposed solution
  detail      VARCHAR2(1000),
  created_at  TIMESTAMP DEFAULT SYSTIMESTAMP
);
```

## Skeleton (Oracle PL/SQL)

```sql
CREATE OR REPLACE PROCEDURE rca_run(p_mode IN VARCHAR2, p_scope IN VARCHAR2 DEFAULT NULL) IS
  v_run NUMBER;
BEGIN
  IF p_mode NOT IN ('CHECK','SIMULATE','EXECUTE') THEN
    RAISE_APPLICATION_ERROR(-20001, 'Invalid mode');
  END IF;
  IF p_mode = 'EXECUTE' THEN
    -- guard: a SIMULATE for this scope must exist
    NULL; -- SELECT ... FROM rca_logs WHERE mode='SIMULATE' AND scope=p_scope AND status='OK'; else RAISE
  END IF;
  -- 1. insert row in rca_logs (autonomous transaction so logs survive rollbacks)
  -- 2. cursor over entities that should have the record and do not
  -- 3. per entity: classify cause by fixed priority; set proposed action
  --      cause with business decision pending  -> state PENDING_DECISION, no action
  --      CHECK                                  -> state DETECTED
  --      SIMULATE                               -> state WOULD_FIX
  --      EXECUTE                                -> perform fix, state FIXED (or ERROR + continue)
  -- 4. close rca_logs with totals
  NULL;
END;
/
```

## Output of a run

Summary by cause and state, and the list of `PENDING_DECISION` entities. Example:

| Cause | Detected | Would fix | Fixed | Pending decision |
|---|---|---|---|---|

## After applying

Re-run the phase 2 sweep for the same pattern (cbm `search_code` / `trace_path`) to find sibling code paths with the same silent-failure shape, and add a log line or explicit error where the code swallowed the condition.
