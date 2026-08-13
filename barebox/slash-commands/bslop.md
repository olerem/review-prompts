Read the prompts {{REVIEW_DIR}}/../kernel/slop-indicators.md and
{{REVIEW_DIR}}/../kernel/subsystem/subjective-review.md

Both are used unchanged; the SLOP-* tells are about prose and code shape, not
about anything Linux-specific.

This is a standalone run of the subjective slop pass (normally part of
agent/review.md's PHASE 4.5, gated by agent/report.md), for testing and
calibration.

Assess the top commit, or the provided patch/commit, for the SLOP-* stylistic tells. Apply the
confidence discipline in {{REVIEW_DIR}}/false-positive-guide.md section 11.1: high bar, cluster
requirement, compare to neighbouring code, defer correctness to a real review, debate yourself,
and a hard cap of 3 observations.

Note that section 11.1 lives in the kernel guide that
{{REVIEW_DIR}}/false-positive-guide.md loads; read the barebox one, which adds
the exceptions.

Output findings as gentle, question-posed comments per
{{REVIEW_DIR}}/../kernel/inline-template.md
("this isn't a bug, but ..."), naming the specific code or prose. Never mention the author and
never imply the code was machine-generated. If nothing clears the bar, say so and emit nothing.

Always load {{REVIEW_DIR}}/deltas.md as well; it overrides the kernel prompts
wherever barebox differs. In particular, do not raise style deviations inside
files imported from Linux (§1.6) or structural suggestions the tree does not
actually follow (§1.7) as slop.
