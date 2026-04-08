# Software Engineer Skills

I'm Vincent, a software engineer. This repo is where I keep the Claude skills that actually make a difference in my day-to-day work — not experiments or one-offs, but the ones I reach for regularly.

## Skills

### [`pr-review-fix`](./pr-review-fix/SKILL.md)
Fetches all unresolved review comments from the current branch's GitHub PR and applies the fixes directly to local files. Handles inline comments, suggested changes, and general review feedback — then summarizes what was changed.

### [`test-gap-finder`](./test-gap-finder/SKILL.md)
Analyzes a git diff (or specific files) and finds untested code paths — error throws, null inputs, branching logic, async failures. Prints a gap report grouped by file, then writes real runnable test stubs (not just TODO scaffolding) in the right test file using whatever framework the project uses.
