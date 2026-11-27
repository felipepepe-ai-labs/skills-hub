# Validation Protocol

How to turn a hypothesis into a test that the **user** runs. Nothing is considered true until a result is pasted back and cross-checked with the code.

## Hard rules for generated tests

- Read-only: `SELECT`, `EXPLAIN`, log greps, HTTP `GET`, running existing tests. Never `INSERT/UPDATE/DELETE/MERGE/TRUNCATE/DDL`, never `COMMIT`.
- Bounded: add a row limit (`FETCH FIRST n ROWS ONLY` / `ROWNUM <= n` / `LIMIT n`) on detail queries; use `COUNT(*)` for totals.
- Parametrized: placeholders (`:param`) for ids and dates, never literals copied from other cases.
- One test per hypothesis, each answering one question. Name tests `T-H<n>`.
- State what each outcome means *before* the user runs it.

## Test template (one per hypothesis)

```
T-H<n> — <hypothesis one-liner>
Objective: <what this proves or refutes>
Run where: <schema / env / log path>
Query / command:
  <read-only statement>
If TRUE expect:  <result shape and values>
If FALSE expect: <result shape and values>
Code cross-check: <file:line the data must be explained by>
```

## Examples (Oracle)

Missing config rows (pattern 1):
```sql
SELECT e.clave
FROM   (SELECT :k1 AS clave FROM dual UNION ALL SELECT :k2 FROM dual) e
LEFT JOIN config_general c ON c.clave = e.clave
WHERE  c.clave IS NULL;
```
Inputs without a mapping (pattern 2):
```sql
SELECT COUNT(*)
FROM   cliente cl
LEFT JOIN cp_agente m ON m.cp = cl.cp
WHERE  m.cp IS NULL;
```
Entities with no active role (pattern 3):
```sql
SELECT p.id
FROM   poliza p
WHERE  NOT EXISTS (SELECT 1 FROM rol r
                   WHERE r.poliza_id = p.id AND r.tipo = 'TITULAR' AND r.fecha_baja IS NULL)
FETCH FIRST 50 ROWS ONLY;
```
Expected-but-missing records (the symptom itself):
```sql
SELECT COUNT(*) FROM poliza p
WHERE  NOT EXISTS (SELECT 1 FROM zzz z WHERE z.poliza_id = p.id);
```

Adapt names to the real schema discovered in phase 2; never run these as written.

## Cross-check with code (second evidence)

After the user pastes data:
1. `get_code_snippet` / Read the exact branch the hypothesis points at.
2. Confirm that the code path **produces** the observed data (e.g. the guard returns when the lookup is empty, and the missing rows are exactly those whose lookup is empty).
3. If counts of "affected by the cause" and "missing in the table" do not reconcile, the difference is another cause: reopen phase 2 for the remainder.

## Verdicts

| Verdict | Condition |
|---|---|
| `CONFIRMED` | Data supports it and code explains it |
| `DISCARDED` | Data contradicts it |
| `INCONCLUSIVE` | Data supports it but code does not explain it, or evidence is missing |

Attribute every affected entity to **one** cause when building the remediation tool (first matching cause in a fixed priority); record overlaps in `trazas`.
