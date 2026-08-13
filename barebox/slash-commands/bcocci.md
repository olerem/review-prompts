Read the prompt `{{REVIEW_DIR}}/../kernel/coccinelle.md`

Generate a Coccinelle semantic patch (.cocci file) for the requested code transformation.

Write the .cocci file and provide a `spatch` command to apply it. Barebox has no
`make coccicheck` target, so give the direct invocation:

    spatch --sp-file <file>.cocci --dir drivers --very-quiet --in-place
