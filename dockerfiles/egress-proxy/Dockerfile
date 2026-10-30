FROM debian:trixie-slim

RUN apt-get update && apt-get upgrade -y \
    && apt-get install -y --no-install-recommends squid \
    && rm -rf /var/lib/apt/lists/*

COPY squid.conf /etc/squid/squid.conf
COPY entrypoint /usr/local/bin/entrypoint

ENTRYPOINT ["/usr/local/bin/entrypoint"]
