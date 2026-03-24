# test-matrix-generator

Scenario-based pytest test matrix scaffolder + coverage checker for Claude Code.

Generates Given/When/Then test structures with variation axes. API call/payload validation is delegated to Postman MCP — this tool focuses on **logic verification**.

## Install

```bash
# Tool
mkdir -p ~/.claude/tools/test-matrix
cp generate.py ~/.claude/tools/test-matrix/

# Skill
mkdir -p ~/.claude/skills/test-matrix
cp skill.md ~/.claude/skills/test-matrix/

# (Optional) Auto-allow in ~/.claude/settings.json permissions.allow:
# "Bash(*python3 ~/.claude/tools/test-matrix/generate.py*)"
```

## Usage

### Claude Code Skill

```
/test-matrix
```

Claude traces the UI flow (component → service → API), defines scenarios, ports logic, and runs tests.

### CLI

#### init — scaffold generation

```bash
cat <<'EOF' | python3 generate.py init -
{
  "name": "bulk-update",
  "output_dir": "tests/bulk_update",
  "scenarios": [
    {
      "id": "bulk_update",
      "desc": "user bulk-updates a list",
      "given": "editor role, 3 items exist",
      "when": "update name field",
      "then": "3 changed, 0 failed",
      "axes": [
        {
          "name": "role",
          "items": [
            {"id": "editor", "desc": "editor role"},
            {"id": "viewer", "desc": "viewer role", "expect_fail": true},
            {"id": "admin", "desc": "admin role"}
          ]
        },
        {
          "name": "item_count",
          "items": [
            {"id": "empty", "desc": "0 items"},
            {"id": "single", "desc": "1 item"},
            {"id": "bulk", "desc": "100 items"}
          ]
        }
      ]
    }
  ]
}
EOF
```

Output:
```
init: tests/bulk_update/
  bulk_update: role(3) x item_count(3) = 9 cases
  total: 9 cases
  files: __init__.py, mocks.py, logic.py, test_scenarios.py
```

#### check — coverage validation

```bash
python3 generate.py check tests/bulk_update/
```

Output:
```
check: tests/bulk_update/
  unfilled:
    ! mocks.py: 6 TODO(s)
    ! logic.py: has NotImplementedError

  axes:
    role: 3 items
    item_count: 3 items

  scenarios:
    standalone: 1 tests, 9 cases (axes: item_count, role)

  summary:
    total tests: 1
    total cases: 9
    parametrized: 1/1
```

## Generated File Structure

```
{output_dir}/
  __init__.py          # Python package
  mocks.py             # Given-state mock data per scenario
  logic.py             # When-step logic stubs (port source code here)
  test_scenarios.py    # @pytest.mark.parametrize scenario tests
```

## Config Spec

```json
{
  "name": "string — test suite name",
  "output_dir": "string — output path (hyphens auto-converted to underscores)",
  "scenarios": [
    {
      "id": "string — scenario identifier",
      "desc": "string — human-readable description",
      "given": "string — preconditions",
      "when": "string — action",
      "then": "string — expected outcome",
      "axes": [
        {
          "name": "string — variation axis name",
          "items": [
            {
              "id": "string — unique identifier",
              "desc": "string — description",
              "expect_fail": "boolean (optional) — if true, tested with pytest.raises"
            }
          ]
        }
      ]
    }
  ]
}
```

- `scenarios` is required. Each must have at least one axis.
- `expect_fail` items validate error paths via `pytest.raises`.
- Axes are combined via cartesian product.

## Workflow

1. **Scenario definition** — Trace UI flow (component → service → API), define Given/When/Then
2. **Logic porting** — Port the When-step source code logic into Python
3. **API inspection** — Use Postman MCP to inspect response/payload structures
4. **Mock data** — Fill Given-state data based on real API responses
5. **Scaffold** — `generate.py init` creates the test structure
6. **Run** — `pytest test_scenarios.py -v`
7. **Check** — `generate.py check` validates coverage and unfilled TODOs

## License

MIT
