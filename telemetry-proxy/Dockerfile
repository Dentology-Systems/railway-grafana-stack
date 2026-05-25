# Pinned (no `latest`). Capitalized so Railway auto-detects the Dockerfile
# builder (no RAILWAY_DOCKERFILE_PATH needed).
ARG VERSION=2.8.4

FROM caddy:${VERSION}-alpine

COPY Caddyfile /etc/caddy/Caddyfile

# Caddy's default entrypoint runs: caddy run --config /etc/caddy/Caddyfile --adapter caddyfile
