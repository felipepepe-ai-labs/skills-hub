# Code Probes

A code probe is a test that **forces the silent branch** and **asserts that the code reports it**. If the code stays silent, the test fails: the failure is the finding.

## Rules

- Never edit production code. Add only new test files, after the user approves them.
- One test per point, named `test_probe_P<n>_<what>` (or the framework equivalent).
- Use fixtures or test doubles. Never point a probe at a real production database.
- The assertion targets the **signal**, not the implementation: a log record with entity id, a raised error, a row in the trace table, a non-zero return code.
- Detect the framework from the repo (`pytest`, `vitest`/`jest`, JUnit, `go test`, utPLSQL). If none exists, say so and propose the smallest option.

## Template

```
Point:    P<n> — <file:line> — <pattern>
Setup:    <state that makes the branch fire: empty lookup, inactive role, missing key>
Act:      <call the function or entry point>
Assert:   <a warning/error/trace exists AND names the entity and the missing thing>
Expected today: RED if silent, GREEN if covered
```

## Forcing each pattern

| Pattern (see catalog) | Setup that triggers it |
|---|---|
| Missing config row | Fixture without the config key |
| Incomplete mapping | Input whose key has no mapping row |
| State filter leaves zero rows | Entity whose only candidate rows are inactive/ended |
| Swallowed exception | Dependency double that raises |
| Empty result treated as OK | Query double returning no rows |
| Early return without log | Inputs that satisfy the guard |
| Format mismatch | Same key as `' 41002 '`, `41002.0`, different case |
| Partial transaction | Failure injected after the first write; assert nothing half-written |

## Examples

Python (pytest, `caplog`):
```python
def test_probe_p1_missing_agent_for_postal_code(caplog, repo_without_mapping):
    result = assign_agent(client_postal_code="99999")
    assert result is None
    assert any("99999" in r.message and r.levelname in ("WARNING", "ERROR") for r in caplog.records)
```
TypeScript (vitest):
```ts
it("probe P2: reports an inactive owner", async () => {
  const warn = vi.spyOn(logger, "warn");
  await createRecord(policyWithOnlyEndedOwner);
  expect(warn).toHaveBeenCalledWith(expect.stringContaining(policyWithOnlyEndedOwner.id));
});
```
PL/SQL (utPLSQL):
```sql
--%test(P3 reports a policy without an active owner)
PROCEDURE probe_p3 IS v_n NUMBER; BEGIN
  pkg_target.process(p_policy_id => 'TEST-NO-OWNER');
  SELECT COUNT(*) INTO v_n FROM trazas WHERE entity_id = 'TEST-NO-OWNER';
  ut.expect(v_n).to_be_greater_than(0);
END;
```

## Inline fallback: swallowed-error targets

When the `silent-failure-hunter` agent is not installed, look for these directly:
1. Empty or context-free catch: `catch {}`, `except: pass`, `WHEN OTHERS THEN NULL`.
2. Dangerous fallbacks: `.catch(() => [])`, `|| default` on operations that can fail.
3. Lost stack traces: `throw new Error(e.message)`, logging then continuing as if OK.
4. Missing async handling: no `await`/`.catch()`, no timeout on network, file or DB calls.
5. Multi-step writes without rollback or compensation.

Report each as a point with `file:line` and feed it into phase 3.
