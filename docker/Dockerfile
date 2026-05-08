# Re-tag the upstream multica-web image into the lazycat registry.
#
# This wrapper does NOT vendor multica source. Multica ships under a
# modified Apache 2.0 license with restrictions on hosted SaaS / commercial
# embedding (see ../LICENSE), so we redistribute only the prebuilt OCI image
# the upstream author publishes to GHCR — no source modification.
#
# The "main" lazycat service is the Next.js web frontend. It already proxies
# /api, /ws, and /auth to the "backend" service via Next.js rewrites
# (REMOTE_API_URL=http://backend:8080 is baked into this image at build time).
#
# Pinned by sha256 digest of multica-web:v0.2.27 for reproducibility.
# Bump by re-resolving with:
#
#   TOKEN=$(curl -fsSL 'https://ghcr.io/token?service=ghcr.io&scope=repository:multica-ai/multica-web:pull' | jq -r .token)
#   curl -sSI "https://ghcr.io/v2/multica-ai/multica-web/manifests/v0.2.27" \
#     -H "Authorization: Bearer $TOKEN" \
#     -H "Accept: application/vnd.oci.image.index.v1+json" \
#     -H "Accept: application/vnd.docker.distribution.manifest.v2+json" \
#     | grep -i digest
FROM ghcr.io/multica-ai/multica-web:v0.2.27@sha256:da1667d15e9d8f7e54098c97a16d9bb1309ddccad52e3404eeae2edf1a1bd638
