# Pinned (no `latest` — that's what broke Tempo). Bump deliberately.
ARG VERSION=2.8.4

FROM caddy:${VERSION}-alpine

COPY Caddyfile /etc/caddy/Caddyfile

# Caddy's default entrypoint runs: caddy run --config /etc/caddy/Caddyfile --adapter caddyfile
