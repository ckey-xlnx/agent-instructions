# Review Instructions

Christopher Key's custom automated review process (ReviewBoard summary, analysis, interactive feedback submission, and learning from feedback) now lives as a Claude Code skill that loads on demand rather than always consuming context:

`.claude/skills/ckey-review/`

Invoke it by asking to review a ReviewBoard request, summarize pending reviews, analyze a review request, or post review feedback. It can also be loaded explicitly when a code-review task requires it.
