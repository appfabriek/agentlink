# syntax=docker/dockerfile:1
# agentlink in a container (docs/docker.md): Debian slim, a user without root,
# tini as PID 1, and agentlink from its release, checked against SHA256SUMS.
#
#   docker build -t agentlink .                  agentlink and Claude Code
#   docker build --target base -t agentlink-base .
#                                                agentlink only, to build on
#   docker build --build-arg AGENTLINK_SOURCE=local \
#     --build-context agentlink-local=<dir> -t agentlink .
#                                                <dir>/agentlink instead of
#                                                the release (a local build)
#
# For linux/amd64 and linux/arm64. Nothing secret is in the image: the invite
# link, the API key and the home of the agent come when it starts.

ARG DEBIAN=bookworm-slim
# Where agentlink comes from: release, or local (with the build context
# agentlink-local). A stage per source, so neither ever comes from the
# cache of the other.
ARG AGENTLINK_SOURCE=release

FROM debian:${DEBIAN} AS fetch
RUN apt-get update \
 && DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends curl ca-certificates \
 && rm -rf /var/lib/apt/lists/*

FROM fetch AS download-agentlink
ARG TARGETARCH
# The release in agentlink/pubspec.yaml (test/docker_test.dart checks it).
ARG AGENTLINK_VERSION=0.18.0
RUN set -eu; \
    case "$TARGETARCH" in \
      arm64) arch=arm64 ;; \
      amd64) arch=x64 ;; \
      *) echo "agentlink draait niet op $TARGETARCH" >&2; exit 1 ;; \
    esac; \
    mkdir -p /out; cd /out; \
    base="https://github.com/appfabriek/agentlink/releases/download/v${AGENTLINK_VERSION}"; \
    curl -fsSLo "agentlink-linux-$arch" "$base/agentlink-linux-$arch"; \
    curl -fsSLo SHA256SUMS "$base/SHA256SUMS"; \
    grep -E "  agentlink-linux-$arch\$" SHA256SUMS | sha256sum -c -; \
    mv "agentlink-linux-$arch" agentlink

FROM scratch AS agentlink-release
COPY --from=download-agentlink /out/agentlink /agentlink

# Empty: --build-context agentlink-local=<dir with agentlink> fills it.
FROM scratch AS agentlink-local

FROM agentlink-${AGENTLINK_SOURCE} AS agentlink

FROM fetch AS fetch-claude
ARG TARGETARCH
# Claude Code, pinned with the SHA-256 from the manifest of that version:
# https://downloads.claude.ai/claude-code-releases/<version>/manifest.json
ARG CLAUDE_VERSION=2.1.322
ARG CLAUDE_SHA256_ARM64=f220ca50ea4f3222043ff83cc224a1262e0196e9897acc0871efa94235b0c1fd
ARG CLAUDE_SHA256_AMD64=bfb3703ad5380242328459bc62ada4bde8ae7ba57c6e1eeadb9c00e08874c478
RUN set -eu; \
    case "$TARGETARCH" in \
      arm64) arch=arm64; sum=$CLAUDE_SHA256_ARM64 ;; \
      amd64) arch=x64; sum=$CLAUDE_SHA256_AMD64 ;; \
      *) echo "Claude Code draait niet op $TARGETARCH" >&2; exit 1 ;; \
    esac; \
    mkdir -p /out; cd /out; \
    curl -fsSLo claude "https://downloads.claude.ai/claude-code-releases/${CLAUDE_VERSION}/linux-$arch/claude"; \
    echo "$sum  claude" | sha256sum -c -; \
    chmod 0755 claude

# agentlink and its start script, without a CLI: the base of an image with
# its own CLI or its own task.
FROM debian:${DEBIAN} AS base
RUN apt-get update \
 && DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends ca-certificates tini \
 && rm -rf /var/lib/apt/lists/* \
 && useradd --uid 10001 --user-group --create-home --home-dir /home/agent --shell /usr/sbin/nologin agent \
 && install -d -o agent -g agent /home/agent/work
COPY --from=agentlink --chmod=0755 /agentlink /usr/local/bin/agentlink
COPY --chmod=0755 agentlink-start /usr/local/bin/
LABEL org.opencontainers.image.title="agentlink" \
      org.opencontainers.image.description="agentlink: koppelt een agent aan een gesprek in een Appfabriek-app" \
      org.opencontainers.image.source="https://github.com/appfabriek/agentlink" \
      org.opencontainers.image.vendor="Appfabriek"
ENV HOME=/home/agent \
    AGENTLINK_HOME=/home/agent/.config/agentlink
USER agent
# The agent's workdir: join records it, run works in it.
WORKDIR /home/agent/work
# The home of the agent: its identity (agent.json) and, with end-to-end
# encryption, its device keys. Losing it means joining again.
VOLUME /home/agent
# Healthy while the server knows the agent: agentlink status exits 0 then
# (not yet joined, a revoked key or no network: 1).
HEALTHCHECK --interval=5m --timeout=30s --start-period=1m --retries=3 CMD ["agentlink", "status"]
# -s: also under an init of its own (docker run --init).
ENTRYPOINT ["/usr/bin/tini", "-s", "--", "agentlink-start"]

# agentlink with Claude Code.
FROM base AS claude
ARG CLAUDE_VERSION=2.1.322
COPY --from=fetch-claude /out/claude /usr/local/bin/
LABEL com.appfabriek.claude.version="${CLAUDE_VERSION}"
ENV CLAUDE_CONFIG_DIR=/home/agent/.claude \
    ANTHROPIC_API_KEY_FILE=/run/secrets/anthropic-api-key \
    DISABLE_AUTOUPDATER=1 \
    CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1 \
    DISABLE_TELEMETRY=1 \
    DISABLE_ERROR_REPORTING=1
