# Style Guide

This repository follows the Zerodrift organization standards with a few skill-specific rules.

## Skill Content

- Write instructions as operational steps that an agent can execute.
- Separate evidence collection, hypothesis, validation, and reporting.
- Avoid unsupported severity claims; require source references or runtime evidence.
- Keep chain-specific guidance in the matching `references/sui/` or `references/aptos/` directory.

## Code And Scripts

- Use POSIX-compatible shell unless a script declares a stricter runtime.
- Use 2-space indentation for JSON, YAML, Markdown examples, and JavaScript.
- Keep generated artifacts out of the repository unless they are fixtures.
- Prefer explicit command examples over prose-only setup notes.

## Markdown

- Use sentence-case headings where possible.
- Keep tables narrow enough to read in GitHub's default view.
- Use fenced code blocks with a language tag when the language is known.
