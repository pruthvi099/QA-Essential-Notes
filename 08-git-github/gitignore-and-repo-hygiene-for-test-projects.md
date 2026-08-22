# .gitignore & Repo Hygiene for Test Projects

## What It Is

A `.gitignore` file tells Git which files/directories to never track — for test automation projects specifically, this means excluding generated artifacts (test reports, traces, screenshots, videos), dependency directories, and sensitive local configuration, none of which belong in version control. This note covers what a well-maintained test project's `.gitignore` should contain and why, extending the artifact concepts from [Tracing, Screenshots & Videos](../02-automation-python-playwright/tracing-screenshots-videos.md) into repo hygiene practice.

## Why It Matters

- Test automation projects generate a *lot* of transient output (traces, videos, HTML reports, screenshots) on every run — accidentally committing these bloats repository size significantly and pollutes diffs with meaningless binary changes on every commit.
- Committing `.env` files or local config containing credentials is a genuine, common security lapse specifically in test projects, since test config often includes API keys or test account credentials that get put in a local file for convenience and then accidentally staged.
- A clean `.gitignore` is one of the first things that signals a well-maintained repository — for a public portfolio repo like this one, sloppy repo hygiene (committed `node_modules`, stray test-result folders) undermines the professional impression the content itself is trying to build.

## How It Works

**What a test automation project's `.gitignore` should typically exclude:**

1. **Dependency directories** — `node_modules/`, Python virtual environments (`venv/`, `.venv/`) — these are reconstructable from `package.json`/`requirements.txt` and shouldn't be committed.
2. **Generated test artifacts** — HTML reports, traces, screenshots, videos (`playwright-report/`, `test-results/`, `*.zip` trace files) — these are regenerated on every run and have no value as committed history.
3. **Local environment/config files** — `.env`, `.env.local` — these often contain credentials or machine-specific settings that must never be committed (see [Secrets & Environment Management in CI](../07-ci-cd/secrets-and-environment-management-in-ci.md)).
4. **IDE/editor-specific files** — `.vscode/` (unless deliberately shared for team config), `.idea/` — personal editor settings that don't belong in a shared repo.
5. **OS-generated files** — `.DS_Store` (macOS), `Thumbs.db` (Windows) — junk files with no project relevance.

## Example

A realistic `.gitignore` for a Python + TypeScript Playwright test automation repository:

```gitignore
# Dependencies
node_modules/
venv/
.venv/
__pycache__/
*.pyc

# Test artifacts — regenerated on every run, never committed
playwright-report/
test-results/
blob-report/
*.zip
screenshots/
videos/
traces/

# Environment and local config — NEVER commit real values
.env
.env.local
.env.*.local

# IDE/editor
.vscode/
.idea/
*.swp

# OS-generated
.DS_Store
Thumbs.db

# Build output
dist/
build/
```

A `.env.example` file (which SHOULD be committed) demonstrating the pattern for sharing required config structure without sharing actual secrets:
```bash
# .env.example — commit this; it documents required vars with NO real values
BASE_URL=https://staging.example.com
API_KEY=your-api-key-here
TEST_ACCOUNT_EMAIL=test@example.com
TEST_ACCOUNT_PASSWORD=your-password-here
```

Checking whether a sensitive file was accidentally already committed in the past (a real, necessary check — adding it to `.gitignore` now doesn't remove it from existing history):
```bash
# Check if .env was ever committed in history
git log --all --full-history -- .env

# If found, it needs to be removed from history entirely (not just
# deleted going forward) — this requires git filter-repo or similar,
# and the exposed credential should be rotated immediately regardless
```

## Production Considerations

- Set up `.gitignore` at the very start of a project, before the first commit — retrofitting it after `node_modules/` or test artifacts have already been committed requires actively removing them from tracking (`git rm -r --cached`), not just adding the ignore rule going forward.
- If a real secret is ever accidentally committed, adding it to `.gitignore` afterward is insufficient — the secret remains in Git history indefinitely and must be treated as compromised (rotate the credential immediately), regardless of whether the file is later removed from tracking.
- Commit a `.env.example` (with placeholder values, as shown) alongside a gitignored `.env` — this documents what configuration a new contributor needs to set up locally, without exposing any real values.

## Common Pitfalls

- Setting up `.gitignore` after test artifacts have already been committed, not realizing the ignore rule only prevents *future* commits — already-tracked files need explicit removal from tracking.
- Assuming that removing a secret from the current file version also removes it from history — anyone can still retrieve the old committed value via `git log`/`git show` unless history itself is rewritten, making rotation the only real remediation for an exposed credential.
- Committing `node_modules/` or similar large, reconstructable dependency directories, bloating repository size and slowing clone times for every future contributor.
- Not committing a `.env.example` alongside a gitignored `.env`, leaving new contributors to guess what environment variables the project actually needs.

## Interview Notes

- Be ready to list what belongs in a `.gitignore` for a test automation project specifically, and explain why each category shouldn't be tracked.
- Understand why adding a secret to `.gitignore` after the fact doesn't remove it from Git history — a common, important security nuance that distinguishes real understanding from surface-level advice.
- Be able to describe the `.env` / `.env.example` pattern and why it's the standard way to document required configuration without exposing real values.

## References

- [GitHub — Ignoring Files](https://docs.github.com/en/get-started/getting-started-with-git/ignoring-files)
- [gitignore.io — Generate .gitignore Files](https://www.toptal.com/developers/gitignore)