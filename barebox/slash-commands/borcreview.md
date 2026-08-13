Read the prompt {{REVIEW_DIR}}/agent/orc.md

That file is the barebox overlay on {{REVIEW_DIR}}/../kernel/agent/orc.md. It
keeps the kernel workflow and adds the barebox overrides — most importantly the
four lines that every spawned agent's prompt must carry, without which the
subagents load the kernel guides and never see deltas.md.

The overlay directory it refers to as <barebox_dir> is {{REVIEW_DIR}}, and
<prompt_dir> is {{REVIEW_DIR}}/../kernel. Both are already absolute; use them
as-is in the agent prompts.

If a git range is provided, it's meant for the false-positive-guide.md section

Using the prompt, do a deep dive regression analysis of the top commit, or the provided patch/commit

Always load {{REVIEW_DIR}}/deltas.md as well; it overrides the kernel prompts
wherever barebox differs.
