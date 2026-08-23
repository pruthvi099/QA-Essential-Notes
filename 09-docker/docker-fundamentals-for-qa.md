# Docker Fundamentals for QA

## What It Is

Docker packages an application (or a test environment) along with everything it needs to run — code, runtime, system libraries, dependencies — into a portable, isolated **container**, built from an **image** defined by a **Dockerfile**. This note covers the core mental model an SDET needs before containerizing a test environment: what an image actually is, how it differs from a running container, and why this solves a real, recurring testing problem.

## Why It Matters

- "It works on my machine but fails in CI" is one of the most common, frustrating testing problems — it's almost always an environment inconsistency (a different OS library version, a different Node/Python version), and Docker directly solves this by making the environment itself part of what's version-controlled and reproducible.
- Test environments (browser binaries, specific language runtime versions, system dependencies) are exactly the kind of thing that drifts between a developer's machine, CI, and staging — containerization eliminates this drift by defining the environment once, explicitly, in a Dockerfile.
- This is foundational to every other note in this folder, and increasingly assumed baseline knowledge for SDET roles at companies with containerized infrastructure — not knowing basic Docker concepts is a real gap in many modern testing environments.

## How It Works

**Core concepts:**
- **Image** — a read-only template containing everything needed to run something (an OS layer, dependencies, application code) — built once from a Dockerfile, then reused to create containers.
- **Container** — a running (or stopped) *instance* of an image — isolated from the host machine and other containers, but lightweight compared to a full virtual machine since it shares the host OS kernel.
- **Dockerfile** — a text file defining, step by step, how to build an image (base image, dependencies to install, files to copy, commands to run).
- **Layers** — each instruction in a Dockerfile creates a cached layer; Docker reuses unchanged layers on rebuild, which is why Dockerfile instruction *order* affects build speed (more on this in [Containerizing Test Environments](./containerizing-test-environments.md)).

**The relationship, in one line:** a Dockerfile defines an image; an image is a template; a container is a running instance of that template — the same image can spin up many identical, isolated containers.

## Example

A minimal Dockerfile and the commands to build and run it, illustrating the image → container relationship directly:

```dockerfile
# Dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt --break-system-packages

COPY . .

CMD ["pytest"]
```

```bash
# Build an IMAGE from the Dockerfile (done once, or whenever the
# Dockerfile/dependencies change)
docker build -t my-test-suite:latest .

# Run a CONTAINER from that image — this can be done repeatedly,
# each time producing a fresh, identical, isolated instance
docker run my-test-suite:latest

# Run it again — a completely separate, independent container,
# but built from the exact SAME image
docker run my-test-suite:latest

# List running containers
docker ps

# See all built images
docker images
```

Demonstrating isolation — a container's filesystem changes don't affect the host or other containers:
```bash
docker run -it my-test-suite:latest bash
# Inside the container:
echo "test" > /app/temp-file.txt
exit

# Back on the host machine — this file does NOT exist here;
# it only existed inside that specific container instance
ls /app/temp-file.txt
# ls: cannot access '/app/temp-file.txt': No such file or directory
```

## Production Considerations

- Use official, minimal base images (`python:3.12-slim` rather than a full OS image) where practical — smaller images build and pull faster, which directly matters for CI pipeline speed (see [Docker in CI Pipelines](./docker-in-ci-pipelines.md)).
- Containers are ephemeral by design — any data written inside a container's filesystem is lost when the container is removed, unless explicitly persisted via a volume (see [Test Data & Volumes in Docker](./test-data-and-volumes-in-docker.md)) — this is a deliberate design property worth understanding before it causes a surprising "where did my data go" moment.
- Pin specific image versions (`python:3.12-slim`, not `python:latest`) — `latest` drifts over time as the base image is updated, silently reintroducing the exact environment-inconsistency problem Docker is meant to solve.

## Common Pitfalls

- Confusing images and containers — an image is a static template; a container is a running instance. "I built a container" (should be "image") and "I have three images running" (should be "containers") are common, telling mix-ups.
- Using `latest` as a base image tag, causing the exact same environment drift over time that containerization was meant to eliminate.
- Assuming data persists inside a container by default — without an explicit volume, anything written during a container's run disappears once that container is removed.
- Not understanding Docker's layer caching, leading to unnecessarily slow rebuilds (addressed in depth in [Containerizing Test Environments](./containerizing-test-environments.md)).

## Interview Notes

- Be ready to explain the image vs. container distinction precisely and cleanly — a very common, foundational Docker interview question.
- Understand why Docker solves the "works on my machine" problem specifically — being able to articulate the *mechanism* (bundling the environment itself, not just the code) shows real understanding, not just familiarity with the buzzword.
- Be able to explain why containers are ephemeral by default and what that means practically for test data that needs to persist.

## References

- [Docker — Get Started](https://docs.docker.com/get-started/)
- [Docker — Dockerfile Reference](https://docs.docker.com/reference/dockerfile/)