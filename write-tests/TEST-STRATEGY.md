# Test Strategy

How to decide what tests to write and at what level. Reference for the write-tests skill.

## The three levels

### Contract tests (primary)

Test a module's public interface. The most important level — these prevent confirmation bias by encoding the behavioral contract before implementation.

**What they test:** Given these inputs to the public interface, expect these outputs/side effects.

**What they don't test:** How the module achieves the result internally.

**Naming:** `test_<behavior>` not `test_<function_name>`. The behavior comes from the bead spec.

```python
# Good — tests behavior
def test_expired_token_is_rejected():
    result = auth.validate_token(expired_token)
    assert result.is_valid is False
    assert result.reason == "expired"

# Bad — tests implementation detail
def test_validate_token_calls_decode_then_check_expiry():
    ...
```

### Integration tests

Test data flow across module boundaries. At least one per feature to verify wiring.

**What they test:** Data enters at module A's interface and arrives correctly at module B.

**When to write them:**
- Feature has beads that depend on each other
- Data transforms as it crosses module boundaries
- Config/input flows through multiple layers to reach the consumer

### Acceptance tests

Map 1:1 to Acceptance Criteria lines in the feature bead. These are the validation contract — what `/validate` checks.

**What they test:** End-to-end user-visible behavior described in plain language.

**Naming:** Use the acceptance criterion text as the test name/docstring.

```python
def test_when_user_signs_up_verification_email_is_sent():
    """When a user signs up, a verification email must be sent within 30 seconds."""
    ...
```

## Stub design principles

### The interface is the contract

Stubs define WHAT the module promises to do (its interface). The implementer decides HOW (the implementation). A stub that constrains HOW is overstepping.

**Good stub — defines interface:**
```python
def calculate_score(features: dict[str, float], weights: WeightConfig) -> Score:
    raise NotImplementedError("stub")
```

**Bad stub — constrains implementation:**
```python
def calculate_score(features: dict[str, float], weights: WeightConfig) -> Score:
    """
    Steps:
    1. Normalize features to [0, 1]
    2. Apply weights via dot product
    3. Clamp result to [0, 100]
    """
    raise NotImplementedError("stub")
```

### Parameter types are interface; return types are contract

- Parameter types: specify in stubs. The caller needs to know what to pass.
- Return types: specify in stubs. The caller needs to know what comes back.
- Internal types (used only within the module): do NOT define in stubs.

### Existing modules: extend, don't replace

When a bead modifies an existing module:
1. Read the existing file.
2. Add new function/method stubs AFTER existing code.
3. Do NOT touch existing function bodies.
4. Import any new types the stubs need.

## Fixture strategy by dependency category

Follows the dependency classification from [DEEPENING.md](../improve-codebase-architecture/DEEPENING.md):

| Category | Test approach |
|---|---|
| **In-process** | No fixture needed. Pure computation — call directly. |
| **Local-substitutable** | Use existing stand-ins (SQLite for Postgres, PGLite, in-memory FS). Reuse project's existing fixtures. |
| **Remote but owned** | In-memory adapter at the port. Test through the interface. |
| **True external** | Mock adapter. Only category where mocks are justified. |

Always check for existing fixtures first. Reuse `conftest.py`, test factories, helpers, test containers. Never create parallel test infrastructure.

## Edge cases

### Project has no test infrastructure

1. Ask the user which test framework to use.
2. Create minimal test config (pytest.ini, jest.config, etc.).
3. Create test directory following framework conventions.
4. Record command: `bd remember test-command "<cmd>"`.

### Tests need database fixtures

1. Check for existing DB test fixtures first.
2. If found: reuse them. Import existing fixtures.
3. If not found: use lightest available approach — in-memory (SQLite), temp files, test containers.
4. Document in the Test Harness section.

### Testing new API endpoints

1. Import app/router from where the bead says it'll be wired.
2. Use project's HTTP test client (TestClient, supertest, httptest).
3. Create route handler stubs that return 501.
4. Test request shapes, response shapes, status codes, headers.

### Bead references existing modules being modified

1. Read existing module and its tests.
2. Identify which existing tests will break (bead should flag these).
3. Write new tests for new behavior only.
4. Store breaking test info via `bd remember test-harness-breaking` for the implementer.
5. Add stub methods to existing module without touching existing code.

### Multiple beads modify the same file

Process beads in dependency order (`bd graph`). Stubs from earlier beads are available to later ones. Each bead's tests import the same module but test different behaviors.
