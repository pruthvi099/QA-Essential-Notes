# Containerizing Test Environments

## What It Is

This note covers writing a real, production-quality Dockerfile specifically for a Playwright/Appium test suite — installing browser/automation dependencies inside the container, optimizing build speed through deliberate layer ordering, and structuring the image so it's fast to build and reliable to run. This builds directly on the image/container/Dockerfile fundamentals from [Docker Fundamentals for QA](./docker-fundamentals-for-qa.md).

## Why It Matters

- Browser automation tools have real system-level dependencies (specific OS libraries Playwright/Selenium need for the browsers to run headlessly) — getting these installed correctly and efficiently inside a container is a genuinely common, practical setup task, not a trivial afterthought.
- Dockerfile layer ordering directly affects CI build speed — a poorly ordered Dockerfile rebuilds far more than necessary on every small code change, adding real, repeated time cost across every CI run.
- This is a concrete, hands-on skill increasingly expected of SDETs working with containerized infrastructure — being able to actually write (not just discuss) a working test-suite Dockerfile is a meaningful, practical differentiator.

## How It Works

**Dockerfile layer caching, and why instruction order matters:** Docker caches each instruction's result as a layer; on rebuild, it reuses cached layers up until the first instruction whose *inputs* have changed, rebuilding everything from that point forward. This means:
- Instructions that change rarely (installing dependencies) should come *before* instructions that change often (copying application/test code) — so a code-only change doesn't force a slow dependency reinstall.
- Copying only the dependency manifest first (`package.json`, `requirements.txt`), installing dependencies, *then* copying the rest of the code is the standard pattern that maximizes cache reuse.

**Playwright-specific considerations:** Playwright provides official pre-built Docker images (`mcr.microsoft.com/playwright`) with all browser dependencies already installed — using these as a base image avoids manually managing the considerable list of OS-level dependencies browsers require to run headlessly in a minimal container.

## Example

**A well-ordered Dockerfile for a TypeScript Playwright suite, using Playwright's official base image:**
```dockerfile
# Official Playwright image — browsers and OS dependencies pre-installed,
# avoiding the need to manually manage a long list of system packages
FROM mcr.microsoft.com/playwright:v1.48.0-jammy

WORKDIR /app

# Copy ONLY the dependency manifest first — this layer only invalidates
# when dependencies actually change, not on every code edit
COPY package.json package-lock.json ./
RUN npm ci

# Copy the rest of the code AFTER dependencies are installed —
# code changes won't force a slow npm ci re-run
COPY . .

CMD ["npx", "playwright", "test"]
```

**Contrast — a poorly ordered Dockerfile that rebuilds dependencies on every code change:**
```dockerfile
FROM mcr.microsoft.com/playwright:v1.48.0-jammy
WORKDIR /app

# BAD: copying everything first means ANY code change invalidates
# this layer, forcing npm ci to re-run on every single build —
# even for a one-line test file edit
COPY . .
RUN npm ci

CMD ["npx", "playwright", "test"]
```

**Python Playwright equivalent, with the same layer-ordering discipline:**
```dockerfile
FROM mcr.microsoft.com/playwright/python:v1.48.0-jammy

WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt --break-system-packages

COPY . .

CMD ["pytest"]
```

**Building and running, showing the cache benefit in practice:**
```bash
docker build -t qa-suite:latest .
# First build: installs all dependencies (slow)

# ... edit a single test file, no dependency changes ...

docker build -t qa-suite:latest .
# Second build: REUSES the cached dependency-install layer,
# only re-copies code and rebuilds from there (fast)
```

## Production Considerations

- Prefer Playwright's official base images over manually installing browser system dependencies on a generic base image — the official images are maintained to track exactly what each Playwright version needs, and manually replicating this list is error-prone and needs ongoing maintenance as Playwright versions change.
- Use a `.dockerignore` file (the Docker equivalent of `.gitignore`, see [.gitignore & Repo Hygiene for Test Projects](../08-git-github/gitignore-and-repo-hygiene-for-test-projects.md)) to exclude `node_modules/`, test artifacts, and `.git/` from the build context — including these unnecessarily slows down the build context upload and can bloat the resulting image.
- Multi-stage builds (using one stage to install dependencies/build, and a leaner final stage to actually run) can further reduce final image size for production deployments, though for test-suite images specifically, the simpler single-stage pattern shown here is usually sufficient since image size matters less than for a deployed application.

## Common Pitfalls

- Copying all application code before installing dependencies, forcing a full dependency reinstall on every single code change — this is the single most common Dockerfile inefficiency, and directly costs real CI time on every build.
- Manually trying to install Playwright's system-level browser dependencies instead of using the official pre-built image, resulting in a fragile, hard-to-maintain list of OS packages that can drift out of sync with what a given Playwright version actually needs.
- Not using a `.dockerignore` file, resulting in an unnecessarily large build context and slower builds as irrelevant files (like `node_modules/` from the host) get sent to the Docker daemon.
- Pinning to a generic base image tag without a specific Playwright version match — mismatched Playwright library and browser-image versions cause confusing runtime errors unrelated to the actual test code.

## Interview Notes

- Be ready to explain Docker layer caching and how Dockerfile instruction order affects build speed — a common, practical Docker interview question, especially relevant for test-suite images that rebuild frequently in CI.
- Understand why using Playwright's official base image is preferable to manually managing browser system dependencies — shows practical, current tooling knowledge.
- Be able to write (or sketch) a correctly-ordered Dockerfile for a test suite from scratch — a common hands-on exercise for roles involving containerized test infrastructure.

## References

- [Playwright — Docker](https://playwright.dev/docs/docker)
- [Docker — Best Practices for Writing Dockerfiles](https://docs.docker.com/build/building/best-practices/)