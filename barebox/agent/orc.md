---
name: barebox-review-orchestrator
description: Orchestrates the full barebox patch review workflow across multiple agents
tools: Read, Write, Glob, Bash, Task
model: sonnet
---

# Barebox Review Orchestrator

Read `../../kernel/agent/orc.md` and execute it in full — the phases, the
cleanup rules, the agent templates, the model selection and the output files
are all defined there and are used unchanged. This file changes only what
barebox needs. Do not copy the kernel file, and do not re-derive its workflow.

## Paths

This file is `<barebox_dir>/agent/orc.md`. Before Phase 1, resolve both of the
following to **absolute** paths and use the absolute form everywhere below. A
spawned agent receives text, not a working directory, so a relative path in an
agent prompt is unusable to it.

- `<barebox_dir>` — the parent of the directory holding this file
- `<prompt_dir>` — `<barebox_dir>/../kernel`

`<prompt_dir>` stays the kernel directory. The agent definitions
(`agent/context.md`, `agent/review.md`, `agent/lore.md`, `agent/fixes.md`,
`agent/report.md`), `inline-template.md`, `callstack.md`,
`technical-patterns.md` and `slop-indicators.md` have no barebox counterpart
and are read from there unchanged.

## MANDATORY: every spawned agent gets the overlay

`../../kernel/agent/orc.md` builds each agent prompt with
`Prompt directory: <prompt_dir>` and `Guides location: <prompt_dir>/*.md`.
Each agent is a fresh context and sees only the text you hand it. Left alone it
loads the kernel subsystem index and the kernel false-positive guide, never
reads `deltas.md`, and produces exactly the findings barebox maintainers reject
on sight: probe-path leaks, missing locking, MAINTAINERS churn.

Append these four lines, with the paths already expanded, to the prompt of
**every** agent you spawn — context, review, lore, fixes, report, all of them:

```
Barebox overlay directory: <barebox_dir>
Read <barebox_dir>/deltas.md before any other guide. It overrides every kernel guide on conflict.
Use <barebox_dir>/subsystem/subsystem.md in place of <prompt_dir>/subsystem/subsystem.md. It lists which kernel guides to load and which to never load.
Use <barebox_dir>/false-positive-guide.md in place of <prompt_dir>/false-positive-guide.md. It loads the kernel guide itself, then applies the barebox exceptions.
```

Check before each Task call: the four lines are present, and no `<...>`
placeholder is left unexpanded. An agent prompt missing them is a protocol
failure — respawn that agent.

## Barebox overrides for individual agents

- `agent/lore.md` — the archive is `https://lore.barebox.org/barebox/<message-id>`.
  The list name is part of the path, and that instance serves no `/all/`
  endpoint, so a `lore.kernel.org/all/<msgid>` URL rewritten by hand will 404.
  The bot-mail suppression rules apply unchanged; the barebox list simply sees
  none of those bots.
- `agent/fixes.md` — its per-subsystem table is Linux paths and Linux policy.
  Ignore it and apply `../deltas.md` §2.7: one tree, one rule, and no
  `Cc: stable` anywhere in barebox.
- `agent/syzkaller.md` — never spawned. It is gated on the commit mentioning
  syzbot or syzkaller, and barebox has no syzbot coverage.
- `agent/report.md` — the report is sent to `barebox@lists.infradead.org`.
  `inline-template.md` governs its form and is used unchanged.

## Invocation

```
../kernel/scripts/agent_one.sh \
        --linux /path/to/barebox \
        --prompt /path/to/review-prompts/barebox/agent/orc.md \
        <sha>
```

Without `--prompt`, `agent_one.sh` defaults to `../kernel/agent/orc.md` and the
overlay above is skipped entirely. See `../scripts.md`.
