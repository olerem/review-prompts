Read the prompt {{REVIEW_DIR}}/review-core.md

If a git range is provided, it's meant for the false-positive-guide.md section

Using the prompt, do a deep dive regression analysis of the top commit, or the provided patch/commit

Always load {{REVIEW_DIR}}/deltas.md as well; it overrides the kernel prompts
wherever barebox differs.
