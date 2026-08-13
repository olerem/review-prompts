---
name: barebox
description: Load anytime the working directory is a barebox tree, and always load it when you answer questions inside the barebox tree.  Barebox knowledge, subsystem specific details, analysis, review, debugging protocols.  Read this anytime you're in the barebox tree
invocation_policy: automatic
---

## ALWAYS READ

1. Load `{{BAREBOX_REVIEW_PROMPTS_DIR}}/deltas.md`

This is the single most important file: barebox shares most of its code and
process with Linux, so the prompts reuse the kernel set, and `deltas.md` records
every place barebox deliberately differs. Without it you will produce findings
barebox maintainers reject on sight (probe-path leaks, locking races, MAINTAINERS
churn).

You consistently skip reading additional prompt files.  These files are
MANDATORY.  This skill exists as a framework for loading additional barebox
prompts.

## Configuration

The review prompts directory is configured during installation:
- **BAREBOX_REVIEW_PROMPTS_DIR**: {{BAREBOX_REVIEW_PROMPTS_DIR}}

This variable is set by the installation script when the skill is installed.

The barebox prompt set is an **overlay** on the kernel prompt set: files it does
not carry are read from `{{BAREBOX_REVIEW_PROMPTS_DIR}}/../kernel/`.
`deltas.md` always wins over anything loaded from there.

## Capabilities

### Patch Review
When asked to review a barebox patch, commit, or series of commits:
1. Load `{{BAREBOX_REVIEW_PROMPTS_DIR}}/review-core.md`
2. Follow the complete review protocol defined there, including the kernel
   files it directs you to load
3. Load subsystem-specific files as directed by
   `{{BAREBOX_REVIEW_PROMPTS_DIR}}/subsystem/subsystem.md`

### Debugging
When asked to debug a barebox crash, hang, warning, or stack trace:
1. Load `{{BAREBOX_REVIEW_PROMPTS_DIR}}/../kernel/debugging.md`
2. Follow the complete debugging protocol defined there
3. Use crash information as entry points into the code analysis

### Coccinelle (Automatic)
When asked to make a code change that is a repeatable pattern across multiple
files (renames, API changes, parameter additions, boilerplate removal, etc.):
1. Load `{{BAREBOX_REVIEW_PROMPTS_DIR}}/../kernel/coccinelle.md`
2. Generate a `.cocci` semantic patch instead of editing files individually
3. Barebox has **no `make coccicheck` target** — it ships `.cocci` files under
   `scripts/coccinelle/` but no rule to run them. Provide a direct `spatch`
   invocation instead:
   `spatch --sp-file <file>.cocci --dir drivers --very-quiet --in-place`

### Subsystem Context
When working on barebox code in specific subsystems:

1. Always read `{{BAREBOX_REVIEW_PROMPTS_DIR}}/deltas.md` first
2. Read `{{BAREBOX_REVIEW_PROMPTS_DIR}}/subsystem/subsystem.md` and load the
   matching guides. It marks which kernel guides to reuse and which to never
   load because barebox has no such subsystem.

## Semcode Integration

When available, use semcode MCP tools for efficient code navigation:
- `find_function` / `find_type`: Get definitions
- `find_callchain`: Trace call relationships
- `find_callers` / `find_calls`: Explore call graphs
- `grep_functions`: Search function bodies
- `diff_functions`: Identify changed functions in patches

## Output

- Patch reviews produce `review-inline.txt` when regressions are found
- Debug sessions produce `debug-report.txt` with analysis results
- Both outputs are formatted for the barebox mailing list
  (`barebox@lists.infradead.org`, plain text, 78 char wrap)
