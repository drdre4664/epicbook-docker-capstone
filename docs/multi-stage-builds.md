# Multi-Stage Builds

A multi-stage Docker build uses **more than one `FROM`** in a single Dockerfile. Each `FROM` starts a fresh image (a “stage”). The final image is the **last** stage — everything in earlier stages stays behind unless you explicitly `COPY --from=...` it forward.

The Dockerfile in `app/` uses two stages:

```dockerfile
# Stage 1 - Builder
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install

# Stage 2 - Runtime
FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY . .
EXPOSE 8080
CMD ["node", "server.js"]
```

## Why Separate Builder From Runtime

The **builder** stage does the messy work: pulling tarballs from npm, compiling any native modules, generating lockfile artifacts. It needs build tools and a writable cache.

The **runtime** stage doesn’t need any of that. It only needs:

- The Node.js binary
- The resolved `node_modules/` directory
- The application source

By starting Stage 2 from a fresh `node:18-alpine` and pulling forward only the resolved `node_modules`, we ship a slim image that contains exactly what production needs to run — and nothing more.

## The Image-Size Benefit

If we baked the whole `npm install` into a single-stage image we’d also ship:

- npm’s download cache
- temporary tarballs
- build artifacts from any native compilation
- the entire npm CLI dependency tree

That’s tens to hundreds of MB of bytes that never run in production. With two stages, those layers are discarded at the stage boundary. Smaller images mean:

- Faster pulls during deploys
- Less disk usage on the host
- Faster cold starts when a node comes up
- Smaller attack surface to scan

## The Security Benefit

Dev dependencies (testing frameworks, build tools, linters, package managers) are a common source of CVE noise. Every package in the runtime image is a package an attacker can potentially exploit if they ever get a foothold.

A multi-stage build means the runtime image **does not contain**:

- `npm` itself (no `npm install` available to an attacker mid-incident)
- devDependencies
- the build cache
- arbitrary tooling that was only needed to *produce* the artifact

Fewer binaries on disk → fewer things to patch → a smaller, harder target. This is the same idea as “distroless” images, applied at the Dockerfile level without changing base images.

## How to Verify

Build both ways and compare:

```bash
docker build -t epicbook-app:multistage ./app
docker images epicbook-app
```

List what’s inside:

```bash
docker history epicbook-app:multistage
docker run --rm epicbook-app:multistage which npm   # should fail / not found
```
