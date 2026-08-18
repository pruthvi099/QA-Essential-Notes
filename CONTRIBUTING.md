 Contributing to QA-Essential-Notes

Thanks for your interest in improving this repo. It's primarily a personal, actively maintained knowledge base, but corrections, suggestions, and genuinely useful additions are welcome.

## Ways to Contribute

### Reporting an issue
If you spot a technical inaccuracy, outdated information, a broken link, or a typo, please [open an issue](../../issues) with:
- The file path of the note in question
- What's incorrect or unclear
- A suggested correction, if you have one

### Suggesting a new topic
If there's a QA/SDET topic you think is missing, open an issue describing:
- The topic
- Which folder it would belong in (see the [Roadmap](./00-start-here/roadmap.md) for the folder structure)
- Why it's worth covering (what gap it fills)

### Submitting a pull request
Small, well-scoped PRs are easiest to review and merge. Before submitting:

1. **Follow the existing note template** — every note follows this structure:
Topic Name
What It Is
Why It Matters
How It Works
Example
Production Considerations
Common Pitfalls
Interview Notes
References
2. **Match the tone** — clear, practical, SDET-to-SDET. No marketing language, no unnecessary filler, no "in today's fast-paced world" style intros.
3. **Code examples must work** — no pseudocode where real code is possible. Verify syntax before submitting.
4. **Use kebab-case filenames**, no dates (e.g., `flaky-test-handling.md`, not `Flaky_Test_Handling_2026.md`).
5. **Cite real, authoritative references** — official documentation only (Playwright docs, MDN, ISTQB, etc.). Never invent a reference.
6. **Update the relevant folder's `README.md`** to link your new note, following the existing format.
7. **Keep quotes/reproduced text minimal** — summarize and paraphrase rather than copying large blocks from other sources.

### What we're not looking for
- Duplicate coverage of an existing topic without a genuinely different angle
- Notes that are just a copy of official documentation with minimal added value
- Overly opinionated content presented as universal fact where multiple valid approaches exist (note trade-offs instead)
- New top-level folders — the repo structure is intentionally fixed; suggest new topics within existing folders unless there's a genuine, discussed long-term gap

## Review Process

PRs are reviewed for technical accuracy, adherence to the note template, and fit with the repo's existing depth and tone. Feedback may be requested before merging — this isn't personal, it's the same bar every note in the repo is held to.

## Code of Conduct

Be respectful and constructive. This is a resource meant to help people learn — keep contributions and discussion focused on that goal.♥️💻🔁