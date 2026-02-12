# Background
We maintain the dev toolchain image at `images/oz-toolchain/Dockerfile` in `oz-images`. This image is used as the base image for Warp/Oz environments.
Docker Scout reports (at least) that the image defaults to root and is missing supply chain attestations.

# Problem
The image currently defaults to root (`USER` not set; `WORKDIR /root`). We want to default to a non-root user, while keeping the image practical for dev workflows (Ruby via rbenv, Node via NodeSource, Postgres/Redis tooling) and compatible with Oz bootstrap scripts.

# Questions and Answers
Q: What should the default user/home/workdir be?
A: Create a dedicated non-root user `oz` with home `/home/oz` and use `/work` as the default working directory (owned by `oz`).

Q: How should Ruby gems/bundler behave under non-root?
A: Keep rbenv and the preinstalled Rubies, but ensure the default user can install gems without root. Set user-writable gem/bundler paths (`GEM_HOME`, `BUNDLE_PATH`) and ensure their `bin/` is on `PATH`.

Q: Should we make `RBENV_ROOT` writable by the non-root user?
A: Yes (for now). In dev images it’s acceptable to `chown -R oz:oz /opt/rbenv` so `rbenv rehash` and future gem installs that add executables are not blocked by permissions.

Q: Do we need `sudo` in the image?
A: Optional. Default non-root typically implies either (a) no need for root, or (b) limited escalations via `sudo`. Since we’re “just playing around” and optimizing for convenience, we can include `sudo` and allow passwordless sudo for `oz`, but we should only do this if we expect workflows to require it.

Q: How do we address “Missing supply chain attestation(s)”?
A: Use BuildKit/buildx attestation flags at build time (provenance + SBOM). Document the canonical build command in `oz-images/README.md`.

# Design
## Non-root defaults
- Add user/group `oz` (UID/GID 1000 unless collision concerns arise).
- Create and own:
  - `/home/oz`
  - `/work`
- Set:
  - `WORKDIR /work`
  - `USER oz`

## Ruby + bundler under non-root
Current state:
- `RBENV_ROOT=/opt/rbenv`
- Rubies installed during build as root.

Adjustments:
- Ensure `oz` can write where runtime installs happen:
  - `ENV GEM_HOME=/home/oz/.gem`
  - `ENV BUNDLE_PATH=/home/oz/.bundle`
  - Add `$GEM_HOME/bin` to `PATH`.
- Change ownership of `$RBENV_ROOT` to `oz` to avoid permission failures from `rbenv rehash` and other rbenv-managed writes.

## Scout/CVEs
- Add a minimal “reduce fixable CVEs” step:
  - `apt-get update && apt-get upgrade -y` during build, then install packages.

## Attestations
- Update `oz-images/README.md` to use buildx multi-arch builds with provenance/SBOM enabled.

# Implementation Plan
1) Update `images/oz-toolchain/Dockerfile`:
   - Add any missing base utilities we’ve already needed (at least `unzip`).
   - Add `apt-get upgrade -y` (kept within the same RUN layer as install to avoid stale lists).
   - Create user `oz`, create `/work`, and `chown` relevant directories.
   - Set `GEM_HOME`, `BUNDLE_PATH`, and extend `PATH`.
   - `USER oz` and `WORKDIR /work` at the end.
2) Update `README.md`:
   - Replace `docker build` with a canonical `docker buildx build --platform linux/amd64,linux/arm64 --push` command.
   - Include provenance/SBOM flags.
3) Rebuild and push (overwrite the existing tag as requested):
   - `touchfuse/oz-toolchain:2026-02-12`
4) Validate locally:
   - Run the built image and confirm the default user is non-root.
   - Smoke check: `ruby -v`, `bundle -v`, `node -v`, `psql --version`.

# Examples
✅ Expected:
- `docker run --rm touchfuse/oz-toolchain:2026-02-12 id -u` returns a non-zero UID.
- `docker run --rm touchfuse/oz-toolchain:2026-02-12 bash -lc 'whoami && pwd'` prints `oz` and `/work`.
- `docker run --rm touchfuse/oz-toolchain:2026-02-12 bash -lc 'gem env home'` resolves under `/home/oz/...`.

❌ Not acceptable:
- Default user is `root`.
- `bundle install` fails due to permissions writing gems.

# Trade-offs
- Chowning `/opt/rbenv` to a non-root user is less “locked down” than root-owned tooling, but improves dev ergonomics and avoids runtime permission failures.
- Adding `apt-get upgrade` increases build time and can reduce reproducibility across rebuilds, but typically reduces “fixable” CVEs for Scout.
- Adding `sudo` (if we do) increases convenience but weakens the non-root boundary; we should only include it if we see real need.

# Implementation Results
## 2026-02-12
Implemented
- Dockerfile now defaults to a non-root user `oz` with `WORKDIR /work`.
- Added `sudo` + `unzip` and configured passwordless sudo for `oz`.
- Set `GEM_HOME=/home/oz/.gem`, `BUNDLE_PATH=/home/oz/.bundle`, and added `$GEM_HOME/bin` to `PATH`.

Build + push
- Built and pushed multi-arch image (amd64/arm64), overwriting `touchfuse/oz-toolchain:2026-02-12`.
- Buildx produced provenance/SBOM attestations.
- Image digest: `sha256:9e789b3275ef85d10f2529fd2f54114bf4cc9f4e66de1ecc665ca7a43ff39de5`.
- Attestation manifest digest: `sha256:ce4ebdababa3bc68da7a3606b20e0cc46eaa489`.

Smoke test
- Note: because this tag was overwritten, local tests should use `docker run --pull=always ...` to avoid using an older locally-cached image.
- Verified via:
  - `docker run --rm --pull=always touchfuse/oz-toolchain:2026-02-12 bash -lc 'whoami; pwd; ruby -v; bundle -v; node -v; psql --version; sudo -n true'`
- Observed:
  - `whoami=oz`, `pwd=/work`
  - Ruby `3.4.8`, Node `v22.22.0`, Postgres client `16.11`
  - `sudo -n true` succeeded (passwordless sudo configured).
