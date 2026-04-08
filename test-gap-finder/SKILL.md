---
name: test-gap-finder
description: >
  Use this skill whenever the user wants to find missing tests, identify untested code paths,
  write test stubs, or improve test coverage for code they've written or changed.
  Trigger on phrases like: "find test gaps", "what am I missing in my tests", "write test stubs
  for this", "check my test coverage", "what should I test here", "I added new code, what tests
  are missing", "generate tests for my changes", "which code paths aren't tested".
  Also trigger when the user shares code or a diff and asks anything about testing it.
  Use this aggressively — if the user mentions tests and new code in the same breath, this skill
  should run.
---

# Test Gap Finder

This skill analyzes code changes (or specific files) and identifies untested logic paths, then offers to write concrete test stubs.

The goal isn't a generic "you should test this function" list — it's a precise map of which specific scenarios are likely to fail silently in production because nobody wrote a test for them. Error paths, edge cases, and branching logic are where bugs live.

## Step 1: Understand the scope

If the user pointed at specific files, use those. Otherwise, pull the diff against the base branch:

```bash
git diff main...HEAD --unified=0
# or if main doesn't exist:
git diff $(git merge-base HEAD origin/HEAD)...HEAD --unified=0
```

If that returns nothing (no changes), check staged files:
```bash
git diff --cached --unified=0
```

Read the full diff carefully. You care about:
- New functions or methods added
- Modified functions (new branches, new parameters, changed logic)
- New classes or modules
- Changes to error handling

## Step 2: Detect the test framework

Look at the project to understand what's in use — don't ask the user:

```bash
# Check package.json for JS/TS
cat package.json 2>/dev/null | grep -E '"jest"|"vitest"|"mocha"|"jasmine"'

# Check for config files
ls jest.config.* vitest.config.* pytest.ini setup.cfg pyproject.toml go.mod 2>/dev/null

# Check existing test files for import patterns
find . -name "*.test.*" -o -name "*.spec.*" -o -name "*_test.*" | head -5 | xargs head -3 2>/dev/null
```

Use whatever you find. If you see both Jest and Vitest, prefer Vitest (it's more modern). If nothing is detected, default to Jest for TypeScript/JS and pytest for Python.

Also identify where tests live — same directory as source, a `__tests__/` folder, a top-level `tests/` directory — so you can put new stubs in the right place.

## Step 3: Find the gaps

For each changed/added piece of code, think through what a thorough reviewer would flag as missing. The most common gaps:

**Error paths** — `try/catch` blocks where only the happy path is tested. What happens when the API call fails? When the DB is down? When the input is malformed?

**Null / undefined / empty inputs** — especially for functions that receive user input or external data. What if the array is empty? The string is blank? The object is missing a required field?

**Branching logic** — every `if/else`, ternary, or `switch` that doesn't have a test for each branch. Focus especially on branches that handle error states or unusual conditions.

**Async edge cases** — promise rejections, timeouts, concurrent calls. Did the change add any async code without testing what happens when it rejects?

**Boundary values** — off-by-one errors, min/max values, zero, negative numbers.

**Side effects** — if a function mutates state, calls an external service, or writes to storage, is there a test that verifies those side effects are (or aren't) triggered?

Don't flag things that are already tested. Check the existing test file for the module before reporting anything.

## Step 4: Print the gap report

Format it so the user can scan it quickly. Group by file.

```
Test gap analysis for 3 changed files:

── src/auth/login.ts ──────────────────────────────────────
  authenticateUser()
  ✗ No test for invalid credentials (returns null from DB)
  ✗ No test for account locked state (user.locked === true)
  ✗ No test for password hashing failure (bcrypt throws)

  refreshToken()
  ✗ No test for expired token (throws TokenExpiredError)
  ✗ No test for malformed token string

── src/billing/invoice.ts ─────────────────────────────────
  generateInvoice()
  ✗ No test for empty line items array
  ✗ No test for currency conversion failure

── src/utils/format.ts ────────────────────────────────────
  formatCurrency()
  ✓ Looks well-tested — skipping

Found 7 gaps across 2 files. Want me to write the test stubs?
```

Be honest about uncertainty — if you're not sure whether something is tested, say "probably missing" rather than asserting it.

## Step 5: Write the stubs (if the user says yes)

When the user confirms, write test stubs that are immediately runnable — not pseudocode. Each stub should:

- Use the correct framework syntax (describe/it for Jest/Vitest, def test_ for pytest)
- Import the function under test correctly (check the existing test file for the import style)
- Set up any obvious mocks or fixtures needed based on the function's dependencies
- Have a clear test name that reads like a sentence ("returns null when user account is locked")
- Leave a `// TODO` comment in the body so the user knows what assertion to fill in

**Jest/Vitest example:**
```typescript
describe('authenticateUser', () => {
  it('returns null when user account is locked', async () => {
    mockDb.findUser.mockResolvedValue({ ...validUser, locked: true });
    // TODO: assert that authenticateUser returns null (or throws)
    const result = await authenticateUser('user@example.com', 'password123');
    expect(result).toBeNull();
  });

  it('throws when bcrypt.compare rejects', async () => {
    mockBcrypt.compare.mockRejectedValue(new Error('bcrypt failure'));
    // TODO: assert that the error propagates or is handled
    await expect(authenticateUser('user@example.com', 'password123')).rejects.toThrow();
  });
});
```

**Pytest example:**
```python
def test_generate_invoice_empty_line_items():
    # TODO: assert behavior when line_items=[]
    with pytest.raises(ValueError):
        generate_invoice(customer_id=1, line_items=[])
```

Write the stubs directly into the appropriate test file. If no test file exists for that module, create one. Follow the naming convention you observed in the project (`auth.test.ts`, `test_auth.py`, etc.).

After writing, print a summary:
```
Written 7 test stubs across 2 files:
  src/auth/__tests__/login.test.ts  (+5 stubs)
  src/billing/__tests__/invoice.test.ts  (+2 stubs)

Run your tests to confirm they're wired up correctly — the stubs are scaffolded but the assertions may need adjusting.
```

## Tips

- Read the existing test file before writing anything — match the style exactly (import style, mock setup, describe nesting, etc.)
- If the function has complex dependencies, stub them out using `vi.mock()` / `jest.mock()` / `pytest.fixture` patterns you see elsewhere in the test suite
- Don't write tests for private/internal helpers unless they're the source of a meaningful bug risk — focus on public API surface
- If a gap looks like it would be hard to test without significant refactoring (tight coupling, no dependency injection), flag it as a note rather than writing a broken stub
