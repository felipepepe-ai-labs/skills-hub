# Data Probes

A data probe is a **read-only query** that counts today the entities that would hit a silent branch. Rules, bounds and verdict logic come from `../root-cause-audit/references/validation-protocol.md`; this file adds the preventive templates.

## Rules

- `SELECT`, `COUNT`, `EXPLAIN` only. No DML, DDL or `COMMIT`.
- Start with `COUNT(*)`; add a bounded sample (`FETCH FIRST 50 ROWS ONLY` / `LIMIT 50`) only if the count is > 0.
- Placeholders for ids and dates. Adapt table and column names to the schema found in phase 2.
- State what a count of 0 and a count > 0 mean before the user runs it.
- Do not paste personal data into reports or Engram; keep counts and identifiers of code.

## Template

```
Point:    P<n>
Question: how many entities would hit this branch today?
Query:    <read-only statement>
Count 0:  LATENT unless the code test is green
Count >0: ACTIVE if the code is silent
```

## Templates by pattern (Oracle)

Config keys the code expects but the table lacks:
```sql
SELECT e.clave
FROM  (SELECT :k1 AS clave FROM dual UNION ALL SELECT :k2 FROM dual) e
LEFT JOIN config_general c ON c.clave = e.clave
WHERE c.clave IS NULL;
```
Population without a mapping row (A → B):
```sql
SELECT COUNT(*) FROM cliente cl
LEFT JOIN cp_agente m ON m.cp = cl.cp
WHERE m.cp IS NULL;
```
Entities with no currently valid role:
```sql
SELECT COUNT(*) FROM poliza p
WHERE NOT EXISTS (SELECT 1 FROM rol r
                  WHERE r.poliza_id = p.id AND r.tipo = 'TITULAR'
                    AND (r.fecha_baja IS NULL OR r.fecha_baja > SYSDATE));
```
Rows an inner join would drop:
```sql
SELECT (SELECT COUNT(*) FROM a) - (SELECT COUNT(*) FROM a JOIN b ON b.k = a.k) AS lost FROM dual;
```
Same key stored in different formats:
```sql
SELECT COUNT(*) FROM cliente WHERE cp <> TRIM(cp) OR LENGTH(cp) <> 5;
```
Expected records already missing (the symptom itself):
```sql
SELECT COUNT(*) FROM poliza p WHERE NOT EXISTS (SELECT 1 FROM zzz z WHERE z.poliza_id = p.id);
```

## Non-SQL variants

Logs: grep for the absence of the expected line per entity. Files and CSV: `awk`/`grep` counting rows with an empty key. HTTP: `GET` on a read-only endpoint listing entities without the expected child. Always read-only.

## Temporal drift

Likelihood grows with time for ended-validity data. Count also entities that will hit the branch soon: `fecha_baja BETWEEN SYSDATE AND SYSDATE + :days`. Report it as a separate line, state `LATENT`.
