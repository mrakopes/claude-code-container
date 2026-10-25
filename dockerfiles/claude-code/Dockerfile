FROM debian:trixie-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates curl git jq less openssh-client \
    && rm -rf /var/lib/apt/lists/*

ARG CLAUDE_VERSION=latest
RUN curl -fsSL https://claude.ai/install.sh | bash -s ${CLAUDE_VERSION} \
    && cp -L /root/.local/bin/claude /usr/local/bin/claude \
    && rm -rf /root/.local

RUN useradd -m -s /bin/bash agent
USER agent
ENV HOME=/home/agent \
    CLAUDE_CONFIG_DIR=/home/agent/.claude \
    DISABLE_AUTOUPDATER=1
