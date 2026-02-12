# oz-images
This repo contains Docker build contexts for Oz/Warp environment base images.

## Layout
- `images/oz-toolchain/Dockerfile`: primary dev toolchain image used by oz-bootstrap environments

## Build + push (manual)
```bash
# example tag format: 2026-02-12
IMAGE=YOUR_DOCKERHUB_USER/oz-toolchain:2026-02-12

docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --provenance=true \
  --sbom=true \
  --push \
  -t "$IMAGE" \
  -f images/oz-toolchain/Dockerfile \
  images/oz-toolchain
```

Notes:
- Keep images public and code-free; private app repos should be cloned at runtime by `oz-bootstrap` using secrets.
