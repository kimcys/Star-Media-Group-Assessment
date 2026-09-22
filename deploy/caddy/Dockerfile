# Stock caddy:2-alpine has no rate-limiting support at all - it's a
# third-party module, not core Caddy - so this builds a Caddy binary
# with github.com/mholt/caddy-ratelimit compiled in, using Caddy's own
# official builder image (xcaddy) rather than hand-rolling a Go build.
FROM caddy:2-builder-alpine AS builder

RUN xcaddy build --with github.com/mholt/caddy-ratelimit

FROM caddy:2-alpine
COPY --from=builder /usr/bin/caddy /usr/bin/caddy
