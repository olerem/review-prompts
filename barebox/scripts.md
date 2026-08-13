# Scripts

There is no `barebox/scripts/`. The kernel helpers in `../kernel/scripts/` are
project-agnostic — they take the tree and the prompt file as arguments — so
they are used unchanged and documented in `../kernel/scripts/scripts.md`.

Both invocations below pass `--prompt`. The defaults point into `../kernel/`
and would run the kernel prompt set against a barebox tree.

Invoke them with barebox paths:

```
# review one commit
../kernel/scripts/review_one.sh \
        --linux /path/to/barebox \
        --prompt /path/to/review-prompts/barebox/review-core.md \
        <sha>

# ORC agent run
../kernel/scripts/agent_one.sh \
        --linux /path/to/barebox \
        --prompt /path/to/review-prompts/barebox/agent/orc.md \
        <sha>
```

`--linux` names the tree to create the review worktree from; it is not
kernel-specific despite the name.

`--prompt` is **not** optional for barebox. `agent_one.sh` defaults to
`../kernel/agent/orc.md`, which resolves its prompt directory to `../kernel/`
and templates that into every subagent prompt it builds. Those agents are fresh
contexts: they would load the kernel subsystem index and the kernel
false-positive guide and never read `deltas.md`. `agent/orc.md` here is a thin
overlay that keeps the kernel workflow and adds the lines each spawned agent
needs.

`create_changes.py`, `claude_xargs.py`, `claude-json.py` and
`find_descendants.py` operate on git ranges and JSON only. Nothing in them
knows what project it is looking at.

## lore-reply

`../kernel/scripts/lore-reply` builds reply mails from a public-inbox archive.
It finds the message with `semcode dig` or `b4 dig`, which read their own
configuration, so it works against the barebox list — but the archive URLs it
prints are the kernel ones.

The barebox archive is `https://lore.barebox.org/barebox/<message-id>`. Note
the list name in the path: unlike lore.kernel.org, this instance serves no
`/all/` endpoint, so the URLs the script prints have to be translated by hand.
