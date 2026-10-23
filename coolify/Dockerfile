# syntax=docker/dockerfile:1.6
#
# Coolify deployment image for vibe-kanban.
#
# The root Dockerfile compiles the Rust server and the frontend from source,
# which exhausts the memory of the Coolify host. This image instead uses the
# prebuilt, statically linked release binary that `npx vibe-kanban` downloads,
# and adds Node + Claude Code so the Claude Code executor works.
#
# To upgrade: take BINARY_TAG / sha256 from
#   https://npm-cdn.vibekanban.com/binaries/<tag>/manifest.json
# (the tag is embedded in the published `vibe-kanban` npm package), and
# CLAUDE_CODE_VERSION from crates/executors/src/executors/claude.rs.

FROM node:24-bookworm-slim

ARG VK_BINARY_TAG=v0.1.44-20260424091429
ARG VK_SHA256=083f15f027a7052916852f40610db34b021f7e7749593623ac3f67a090447a6b
ARG CLAUDE_CODE_VERSION=2.1.119

RUN apt-get update \
  && apt-get install -y --no-install-recommends \
    ca-certificates \
    curl \
    gh \
    git \
    jq \
    openssh-client \
    ripgrep \
    tini \
    unzip \
    wget \
  && rm -rf /var/lib/apt/lists/*

RUN curl -fsSL -o /tmp/vk.zip \
    "https://npm-cdn.vibekanban.com/binaries/${VK_BINARY_TAG}/linux-x64/vibe-kanban.zip" \
  && echo "${VK_SHA256}  /tmp/vk.zip" | sha256sum -c - \
  && unzip -q /tmp/vk.zip -d /usr/local/bin \
  && chmod 755 /usr/local/bin/vibe-kanban \
  && rm /tmp/vk.zip

# `claude` on PATH for manual use (e.g. `claude setup-token` from a terminal).
RUN npm install -g "@anthropic-ai/claude-code@${CLAUDE_CODE_VERSION}" \
  && npm cache clean --force

# Run as the non-root `node` user: Claude Code refuses
# --dangerously-skip-permissions as root. Mount a persistent volume on
# /home/node — it holds the vibe-kanban DB/config (~/.local/share/vibe-kanban),
# Claude Code state (~/.claude), the npx cache and the cloned repos (~/repos).
USER node
WORKDIR /home/node/repos

# Warm the npx cache: vibe-kanban launches `npx -y @anthropic-ai/claude-code@<ver>`.
RUN npx -y "@anthropic-ai/claude-code@${CLAUDE_CODE_VERSION}" --version \
  && mkdir -p /home/node/.local/share/vibe-kanban /home/node/.claude \
  && git config --global user.name "Vibe Kanban" \
  && git config --global user.email "vibe-kanban@users.noreply.github.com" \
  && git config --global init.defaultBranch main

ENV HOST=0.0.0.0 \
    PORT=3000 \
    RUST_LOG=info

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=5s --start-period=30s --retries=3 \
  CMD ["wget", "--spider", "-q", "http://127.0.0.1:3000/api/health"]

ENTRYPOINT ["/usr/bin/tini", "--", "/usr/local/bin/vibe-kanban"]
