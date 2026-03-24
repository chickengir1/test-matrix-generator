# Test Matrix Generator

Generate scenario-based pytest test matrices. Focuses on logic verification; API call/payload validation is handled by Postman MCP.

## Trigger

`/test-matrix`

## Instructions

### Step 1: Scenario Definition

Ask the user:
- "What feature do you want to test? Any related source files?"

Then **autonomously** trace the UI flow in the codebase:
- Start from the component (UI entry point)
- Follow the code path: component → service → API call
- Identify branching conditions, state transformations, and error handling at each step
- Use Explore agent for narrow scope, Agent Teams for wide scope

Based on the traced flow, **define scenarios** in Given/When/Then format:
- Each scenario maps to a real user journey (e.g., "user opens list → edits cell → saves")
- Given = preconditions (role, data state)
- When = the action chain through the UI flow
- Then = expected outcome
- Mark expected-failure variations (e.g., unauthorized role, empty input)

Present the scenarios to the user for confirmation.

### Step 2: Logic Porting

Port the When-step logic from the source code into Python:
- Reproduce the component/service logic that runs between user action and API call
- Include data transformations, validations, and branching
- This is the core of what the test matrix verifies

### Step 3: API Response Structure (Postman MCP)

Ask the user which Postman workspace/collection has the relevant endpoints, then:
- Look up related requests/responses from Postman collections
- For GET endpoints: inspect response shape (fields, types, nullability, arrays)
- For POST/PATCH endpoints: inspect request payload structure and response
- Use this data as the basis for realistic mock data

Skip this step for pure logic tests.

### Step 4: Mock Data

Fill mocks.py with Given-state data:
- Base field values on Postman response shapes from Step 3
- Each axis item should represent a realistic variation
- Items with `expect_fail: true` represent error paths

### Step 5: Scaffold Generation

Compose config JSON and invoke the scaffolder:

```bash
cat <<'EOF' | python3 ~/.claude/tools/test-matrix/generate.py init -
{
  "name": "project_name",
  "output_dir": "{repo_root}/.claude/tests/{topic}",
  "scenarios": [
    {
      "id": "scenario_id",
      "desc": "user does X",
      "given": "preconditions",
      "when": "action chain",
      "then": "expected outcome",
      "axes": [
        {
          "name": "axis_name",
          "items": [
            {"id": "normal", "desc": "normal case"},
            {"id": "error", "desc": "error case", "expect_fail": true}
          ]
        }
      ]
    }
  ]
}
EOF
```

Then fill the generated skeleton with logic from Step 2 and mocks from Step 4.

### Step 6: Run + Verify

```bash
cd {repo_root}/.claude && python3 -m pytest tests/{topic}/test_scenarios.py -v
```

Check coverage:

```bash
python3 ~/.claude/tools/test-matrix/generate.py check {repo_root}/.claude/tests/{topic}/
```

Reports: pass/fail/skip counts, unfilled TODOs, coverage per scenario.

## Rules

- Scenario definition is the most important step. Trace the full UI flow before defining scenarios
- Do not generate code before understanding the component → service → API code path
- Scenarios must be confirmed by the user before generation
- Mock data must reflect real production data structures, grounded in Postman response shapes
- Always fill content after scaffolding (never leave TODOs)
- API call/payload validation is delegated to Postman MCP. Test matrix focuses solely on logic verification
- Items with `expect_fail: true` are tested with `pytest.raises`
