Read the prompt {{REVIEW_DIR}}/review-core.md first — it is the barebox entry
point and it loads the kernel protocol and deltas.md for you.

Then read {{REVIEW_DIR}}/../kernel/slash-commands/kseries.md for the
series-review workflow, which is used unchanged. Ignore its first line: the
prompt it tells you to load is already loaded, and loading the kernel
review-core.md directly would drop the barebox overrides.

Barebox differences:

- The archive is https://lore.barebox.org/barebox/<message-id> — the list name
  is part of the path and there is no /all/ endpoint on that instance.
- The list is barebox@lists.infradead.org.

Always load {{REVIEW_DIR}}/deltas.md as well; it overrides the kernel prompts
wherever barebox differs.
