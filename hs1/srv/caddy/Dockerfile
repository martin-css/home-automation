ARG CADDY_VERSION=2.11.4

FROM caddy:${CADDY_VERSION}-builder-alpine AS builder

ARG CADDY_DNS_DIGITALOCEAN_COMMIT=04bde2867106aa1b44c2f9da41a285fa02e629c5
RUN xcaddy build ${CADDY_VERSION} \
    --with github.com/caddy-dns/digitalocean@${CADDY_DNS_DIGITALOCEAN_COMMIT}

FROM caddy:${CADDY_VERSION}-alpine
COPY --from=builder /usr/bin/caddy /usr/bin/caddy
