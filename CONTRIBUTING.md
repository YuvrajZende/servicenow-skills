# Contributing to sn-architect

Thanks for helping make AI agents better ServiceNow architects. This project has one non-negotiable standard:

## No invented facts

Every factual claim — CLI commands, API signatures, table names, ACL behavior — must trace to a source: official docs, the official SDK CLI reference (servicenow.github.io/sdk/cli), the official examples repo, or a ServiceNow Support article. Corrections **must include the link**. PRs adding unverifiable claims will be declined.

## Highest-value contributions

1. **Failing test scenarios** — requirements where agents *without* the skill produce broken designs (extend `sys_user`, empty ACLs, sync REST in transactions, custom approval tables...). Include the prompt and, if possible, the baseline output.
2. **Corrections with sources** — anything factually wrong, with the official link.
3. **New reference files** — follow the house format:
   - Title + one-line blockquote purpose
   - Decision matrix (tables, not essays)
   - Hard rules with falsification tests ("**The test:** ...")
   - Anti-patterns section
   - "Output in the Architecture Document" section when the concern produces a deliverable
   - "Authoritative Sources" with official links
4. **Behavioral test evidence** — RED/GREEN runs (with vs without the skill) are gold; see the README's Test evidence section for format.

## House style

- Decision matrices over prose. Concise beats complete.
- Every rule gets a one-line falsification test an agent can self-check.
- Cross-reference other references by filename (`see \`05-security-design.md\``) instead of repeating content.
- Mark unverified claims explicitly — never let them look authoritative.

## Testing your change

Reference edits should be retrieval-tested: with only `SKILL.md` in context, a fresh agent should route your concern to the right file, and the file should answer the question without fabrication. If your change adds new metadata claims, verify against `npx @servicenow/sdk explain <topic>` or official docs first.

## License

By contributing, you agree your contributions are licensed under the MIT License of this repository.