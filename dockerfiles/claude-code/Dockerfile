FROM debian:trixie-slim

# bumping the version invalidates all layers below -> fresh apt upgrade too
# (bin/ccd-update checks for a new release, bumps it and rebuilds)
ARG CLAUDE_VERSION=2.1.314

RUN echo "claude-code ${CLAUDE_VERSION}" \
    && apt-get update && apt-get upgrade -y \
    && apt-get install -y --no-install-recommends ca-certificates curl \
    && rm -rf /var/lib/apt/lists/*

RUN curl -fsSL https://claude.ai/install.sh | bash -s ${CLAUDE_VERSION} \
    && cp -L /root/.local/bin/claude /usr/local/bin/claude \
    && rm -rf /root/.local

RUN apt-get update && apt-get install -y --no-install-recommends \
      git jq less openssh-client python3 python3-pip python3-venv vim \
    && rm -rf /var/lib/apt/lists/*

RUN curl -fsSL https://packages.buildkite.com/helm-linux/helm-debian/gpgkey \
      -o /usr/share/keyrings/helm.asc \
    && echo "deb [signed-by=/usr/share/keyrings/helm.asc] https://packages.buildkite.com/helm-linux/helm-debian/any/ any main" \
      > /etc/apt/sources.list.d/helm.list \
    && apt-get update && apt-get install -y --no-install-recommends helm \
    && rm -rf /var/lib/apt/lists/*

RUN curl -fsSL https://apt.releases.hashicorp.com/gpg \
      -o /usr/share/keyrings/hashicorp.asc \
    && echo "deb [signed-by=/usr/share/keyrings/hashicorp.asc] https://apt.releases.hashicorp.com trixie main" \
      > /etc/apt/sources.list.d/hashicorp.list \
    && apt-get update && apt-get install -y --no-install-recommends terraform \
    && rm -rf /var/lib/apt/lists/*

COPY ccd-statusline /usr/local/bin/ccd-statusline
COPY managed-settings.json /etc/claude-code/managed-settings.json

RUN useradd -m -s /bin/bash agent
USER agent
ENV HOME=/home/agent \
    CLAUDE_CONFIG_DIR=/home/agent/.claude \
    DISABLE_AUTOUPDATER=1 \
    EDITOR=vim

LABEL claude-code.version=${CLAUDE_VERSION}
