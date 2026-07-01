# Bead Spec Format

How to write implementation-ready bead descriptions. Beads are the single source of truth for specifications — there are no separate brief files.

## Two bead types per feature

### Feature bead (one per feature)

Created with `--type feature`. Contains the high-level goal, context, and acceptance criteria. Task beads reference this for the big picture.

Use a heredoc for descriptions to avoid permission prompts from `#` characters:

```bash
bd create --type feature --priority 2 \
  --title "Add email verification on signup" \
  --description "$(cat <<'BEAD'
Goal
Email verification prevents fake accounts and ensures deliverability.

Context files
- CONTEXT.md — domain glossary and architecture
- src/auth/models.py — existing User model
- src/routes/signup.py — current signup flow

Acceptance Criteria
- When a user signs up with a valid email, a verification email is sent within 30 seconds
- When a user clicks the verification link before expiry, their account is marked verified
- When a user clicks an expired verification link, they see an error with a resend option
- Existing signup flow for already-verified paths produces identical behavior to before this change
BEAD
)"
```

**Do NOT use `#` headers inside bead descriptions.** Use plain text section labels instead. The `#` character inside quoted shell arguments triggers security validation prompts.

**Acceptance Criteria are mandatory.** Every feature bead must include behavioral "when X then Y" assertions that are testable, implementation-independent, and cover every user-visible outcome. These are what `/write-tests` turns into failing tests and what `/validate` checks against.

Each criterion must be:
- Testable without reading the implementation
- Specific enough to write a pass/fail assertion
- Independent of internal structure (no "function X calls Y")

NOT acceptable:
- "Verify it works" — not testable
- "Check the response" — no expected value
- "Run make unit-test" — that's a verification step, not a criterion

### Task beads (one per unit of work)

Created with `--type task`. Contains the implementation spec for one bead of work.

```bash
bd create --type task --priority 2 \
  --title "Add TokenService with verification token generation" \
  --description "$(cat <<'BEAD'
Files
src/auth/tokens.py (new)
src/auth/models.py (modify — add verification_token field)

What to do
1. Add VerificationToken model to src/auth/models.py — fields: token (str), user_id (int), expires_at (datetime), used (bool)
2. Add TokenService class to src/auth/tokens.py with:
   - generate_token(user_id: int) -> VerificationToken — creates token with 24h expiry
   - validate_token(token: str) -> TokenResult — returns valid/expired/invalid status
3. Add VerificationToken to the User relationship in src/auth/models.py

Design decisions
- "Use cryptographic random tokens, not JWT" — user decision during grilling
- "Token expiry: 24 hours" — per domain requirements
- "One active token per user" — previous tokens invalidated on new generation

Test expectations
- Contract: generate_token produces unique tokens with correct expiry
- Contract: validate_token with valid/expired/used/malformed tokens
- Edge: generating new token invalidates previous ones
BEAD
)"
```

## Rules

- **Be concrete.** File paths, class names, method signatures. Not "update the service" but "add `calculate_discount(order: Order, rules: DiscountRules) -> Discount` to `OrderProcessor` in `src/orders/processor.py`."
- **Relative paths only.** All file paths in bead descriptions MUST be relative to the repo root (e.g. `src/auth/models.py`). NEVER use absolute paths (e.g. `/Users/joe/project/src/auth/models.py`). Agents may work in git worktrees where absolute paths point to the wrong directory.
- **No code bodies.** Specify function signatures (name, params, return type) and behavioral contracts ("given X input, returns Y"). Never write method implementations, class bodies, or algorithm steps in code blocks. The bead defines the WHAT and the interface — the implementer and test writer decide the HOW.
- **Include current state.** If building on prior work, say what exists and what's remaining.
- **Quote design decisions.** The coding agent wasn't in the grilling session. It needs the WHY, not just the WHAT.
- **No ambiguity.** If there are two valid approaches, the planner already picked one. The bead states which.

## Execution order

Modeled entirely by bead dependencies. **Direction matters and the two command forms take reversed argument orders** — use the `--blocks` form, which reads left-to-right:

```
bd dep <blocker-id> --blocks <blocked-id>    # blocker must be done before blocked can start
bd dep add <blocked-id> <blocker-id>         # SAME edge, REVERSED args — easy to get wrong
```

The foundation (model/shared types) is the blocker; the feature bead is blocked by everything. After wiring, verify with `bd dep cycles` (no cycles), `bd dep tree <feature-id>` (foundation at the bottom), and `bd ready` (only the intended starting bead is ready). `bd graph` shows the full dependency tree.

## Test Harness (added by `/write-tests`)

After `/write-tests` runs, it creates a **test harness bead** (`--type task`) containing stub file paths, test file paths, baseline test count, and breaking test notes. This bead is wired as a dependency of the feature bead so the implementer sees it via `bd graph`.

The implementer checks for a test harness bead by looking for a bead titled "Test harness for ..." in `bd list --status=open`.

## Constraints

Standard constraints for implementers (stored once, not per-bead):
- Use the project's configured runner (check `CONTEXT.md` or `bd recall runner`)
- Imports at top of files, never inside functions
- `__init__.py` files: only re-exports, never definitions
- Run tests using the project's test command (check `CONTEXT.md` or `bd recall test-command`)
- `bd update <id> --claim` when starting, `bd close <id>` when done
