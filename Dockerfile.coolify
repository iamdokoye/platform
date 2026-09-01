# syntax=docker/dockerfile:1.7
# ================================================================
# Promaly Platform — Coolify / self-hosted build Dockerfile
#
# Usage (via docker-compose.coolify.yml):
#   Each service is a named stage here; docker-compose selects it
#   with `target: <stage-name>`.
#
# Structure:
#   builder        — full Rush monorepo compile + esbuild bundles
#   rt-slim        — shared slim Node runtime (reusable layer)
#   rt-full        — shared full Node runtime with native modules
#   preview-base   — Node + ffmpeg + LibreOffice (for preview service)
#   rekoni-base    — Node + document converters (for rekoni)
#   print-base     — Node + Chromium (for print service)
#   account        — account service
#   stats          — stats/metrics service
#   transactor     — real-time transactor (pod-server)
#   workspace      — workspace manager
#   collaborator   — collaborative editing
#   front          — web frontend server
#   fulltext       — full-text search indexer
#   datalake       — blob / file storage gateway
#   hulylake       — (built separately, see foundations/hulylake/Dockerfile)
#   preview        — file preview generator
#   link-preview   — URL link previews
#   rekoni         — document content extraction
#   export         — workspace export service
#   analytics      — analytics collector
#   backup         — workspace backup agent
#   backup-api     — backup REST API
#   process-svc    — process automation
#   time-machine   — scheduled worker
#   events-proc    — event stream processor
#   rating         — rating / reputation service
#   sign           — document signing (optional)
#   print          — PDF print renderer / Chromium (optional, heavy)
# ================================================================

ARG NODE_VERSION=22

# ──────────────────────────────────────────────────────────────
# Stage: builder
# Installs Rush + pnpm, then compiles and bundles the full
# monorepo.  All subsequent stages COPY from here.
# ──────────────────────────────────────────────────────────────
FROM node:${NODE_VERSION}-bookworm AS builder

RUN apt-get update && apt-get install -y --no-install-recommends \
        git python3 make g++ \
    && rm -rf /var/lib/apt/lists/*

# Rush version must match rushVersion in rush.json (5.158.1)
RUN npm install -g @microsoft/rush@5.158.1

WORKDIR /app

# Copy entire platform tree
COPY . .

# Silence git safe-directory warnings inside Docker
RUN git config --global --add safe.directory /app 2>/dev/null || true

# Install all workspace dependencies (Rush manages pnpm internally)
RUN rush install

# ── Phase 1: esbuild-bundle all services ──
# rush bundle runs _phase:build (TypeScript) + _phase:bundle (esbuild)
# -p 20 allows up to 20 parallel workers
RUN rush bundle -p 20 \
    --to @hcengineering/pod-account \
    --to @hcengineering/pod-server \
    --to @hcengineering/pod-workspace \
    --to @hcengineering/pod-collaborator \
    --to @hcengineering/pod-stats \
    --to @hcengineering/pod-fulltext \
    --to @hcengineering/pod-preview \
    --to @hcengineering/pod-link-preview \
    --to @hcengineering/pod-backup \
    --to @hcengineering/pod-datalake \
    --to @hcengineering/pod-export \
    --to @hcengineering/pod-analytics-collector \
    --to @hcengineering/rekoni-service \
    --to @hcengineering/pod-sign \
    --to @hcengineering/backup-api-pod \
    --to @hcengineering/pod-process \
    --to @hcengineering/pod-worker \
    --to @hcengineering/pod-events-processor \
    --to @hcengineering/pod-rating \
    --to @hcengineering/pod-front

# ── Phase 2: webpack UI assets + copy into pods/front/dist ──
# rush package runs _phase:build + _phase:package.
# For dev/prod: webpack build → dist/
# For pod-front: copies that dist into pods/front/dist/
# Rush incremental cache skips already-compiled TS from phase 1.
RUN rush package --to @hcengineering/pod-front


# ──────────────────────────────────────────────────────────────
# Shared runtime base layers (match dev/base-image/ definitions)
# ──────────────────────────────────────────────────────────────

# Slim base: libjemalloc + dumb-init, no native npm modules
FROM node:${NODE_VERSION}-slim AS rt-slim
RUN apt-get update && apt-get install -y --no-install-recommends \
        libjemalloc2 dumb-init ca-certificates \
    && apt-get clean && rm -rf /var/lib/apt/lists/*
ENV LD_PRELOAD=libjemalloc.so.2 \
    MALLOC_CONF=dirty_decay_ms:1000,narenas:2,background_thread:true \
    NODE_ENV=production
WORKDIR /usr/src/app

# Full base: also pre-installs native Node add-ons (sharp, snappy, bufferutil)
FROM node:${NODE_VERSION} AS rt-full
RUN apt-get update && apt-get install -y --no-install-recommends \
        libjemalloc2 dumb-init ca-certificates \
    && apt-get clean && rm -rf /var/lib/apt/lists/*
ENV LD_PRELOAD=libjemalloc.so.2 \
    MALLOC_CONF=dirty_decay_ms:1000,narenas:2,background_thread:true \
    NODE_ENV=production
WORKDIR /app
RUN npm install --ignore-scripts=false bufferutil sharp@v0.34.3 utf-8-validate snappy --unsafe-perm

# Preview base: full Node + ffmpeg, poppler, LibreOffice
FROM node:${NODE_VERSION} AS preview-base
RUN apt-get update && apt-get install -y --no-install-recommends \
        dumb-init libjemalloc2 ffmpeg poppler-utils libreoffice \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* /usr/share/doc/* /usr/share/man/*
ENV LD_PRELOAD=libjemalloc.so.2 \
    MALLOC_CONF=dirty_decay_ms:1000,narenas:2,background_thread:true \
    NODE_ENV=production
RUN npm install --ignore-scripts=false bufferutil sharp@v0.34.3 utf-8-validate snappy --unsafe-perm
WORKDIR /app

# Rekoni base: slim Node + document-format converters + pdfjs
FROM rt-slim AS rekoni-base
RUN apt-get update && apt-get install -y --no-install-recommends \
        coreutils antiword poppler-utils html2text unrtf \
    && rm -rf /var/lib/apt/lists/*
RUN npm install --ignore-scripts=false sharp@v0.34.3 pdfjs-dist@v2.12.313 --unsafe-perm

# Print base: full Node + Chromium (for Puppeteer PDF rendering)
ARG CHROMIUM_VERSION=139.0.7258.154-1~deb12u1
FROM rt-full AS print-base
# Chromium conflicts with jemalloc preload
ENV PUPPETEER_SKIP_CHROMIUM_DOWNLOAD=true \
    PUPPETEER_EXECUTABLE_PATH=/usr/bin/chromium \
    LD_PRELOAD= \
    MALLOC_CONF=
RUN apt-get update && apt-get install -y --no-install-recommends \
        gnupg wget libxss1 \
        fonts-ipafont-gothic fonts-wqy-zenhei fonts-thai-tlwg \
        fonts-kacst fonts-freefont-ttf \
    && apt-get install -y --no-install-recommends \
        chromium-common=${CHROMIUM_VERSION} \
        chromium=${CHROMIUM_VERSION} \
    && apt-get clean && rm -rf /var/lib/apt/lists/*


# ──────────────────────────────────────────────────────────────
# Service stages
# ──────────────────────────────────────────────────────────────

FROM rt-slim AS account
COPY --from=builder /app/pods/account/bundle/bundle.js ./
EXPOSE 3000
CMD ["node", "bundle.js"]

FROM rt-slim AS stats
COPY --from=builder /app/pods/stats/bundle/bundle.js ./
EXPOSE 4900
CMD ["node", "bundle.js"]

FROM rt-slim AS transactor
COPY --from=builder /app/pods/server/bundle/bundle.js \
                    /app/pods/server/bundle/bundle.js.map \
                    /app/pods/server/bundle/model.json ./
EXPOSE 3332
CMD ["node", "./bundle.js"]

FROM rt-slim AS workspace
COPY --from=builder /app/pods/workspace/bundle/bundle.js \
                    /app/pods/workspace/bundle/bundle.js.map ./
CMD ["node", "bundle.js"]

FROM rt-slim AS collaborator
COPY --from=builder /app/pods/collaborator/bundle/bundle.js \
                    /app/pods/collaborator/bundle/bundle.js.map ./
EXPOSE 3078
CMD ["node", "bundle.js"]

FROM rt-full AS front
COPY --from=builder /app/pods/front/bundle/bundle.js \
                    /app/pods/front/bundle/bundle.js.map ./
COPY --from=builder /app/pods/front/dist/ ./dist/
EXPOSE 8080
CMD ["node", "./bundle.js"]

FROM rt-slim AS fulltext
COPY --from=builder /app/pods/fulltext/bundle/bundle.js \
                    /app/pods/fulltext/bundle/bundle.js.map \
                    /app/pods/fulltext/bundle/model.json ./
EXPOSE 4702
CMD ["node", "--expose-gc", "bundle.js"]

FROM rt-slim AS datalake
WORKDIR /app
COPY --from=builder /app/services/datalake/pod-datalake/bundle/bundle.js \
                    /app/services/datalake/pod-datalake/bundle/bundle.js.map ./
CMD ["dumb-init", "node", "bundle.js"]

FROM preview-base AS preview
COPY --from=builder /app/pods/preview/bundle/bundle.js \
                    /app/pods/preview/bundle/bundle.js.map ./
EXPOSE 4040
CMD ["dumb-init", "node", "bundle.js"]

FROM rt-slim AS link-preview
WORKDIR /app
COPY --from=builder /app/pods/link-preview/bundle/bundle.js \
                    /app/pods/link-preview/bundle/bundle.js.map ./
EXPOSE 4041
CMD ["dumb-init", "node", "bundle.js"]

FROM rekoni-base AS rekoni
COPY --from=builder /app/services/rekoni/bundle/bundle.js ./
EXPOSE 4004
CMD ["node", "./bundle.js"]

FROM rt-slim AS export-svc
WORKDIR /app
COPY --from=builder /app/services/export/pod-export/bundle/bundle.js ./
EXPOSE 4009
CMD ["dumb-init", "node", "bundle.js"]

FROM rt-slim AS analytics
COPY --from=builder /app/services/analytics-collector/pod-analytics-collector/bundle/bundle.js ./
RUN mkdir -p /usr/src/geodb
EXPOSE 4017
CMD ["node", "bundle.js"]

FROM rt-slim AS backup
COPY --from=builder /app/pods/backup/bundle/bundle.js \
                    /app/pods/backup/bundle/bundle.js.map \
                    /app/pods/backup/bundle/model.json ./
CMD ["node", "--expose-gc", "bundle.js"]

FROM rt-slim AS backup-api
WORKDIR /app
COPY --from=builder /app/services/backup/backup-api-pod/bundle/bundle.js \
                    /app/services/backup/backup-api-pod/bundle/bundle.js.map ./
EXPOSE 4039
CMD ["dumb-init", "node", "bundle.js"]

FROM rt-slim AS process-svc
COPY --from=builder /app/services/process/bundle/bundle.js ./
CMD ["node", "bundle.js"]

FROM rt-slim AS time-machine
COPY --from=builder /app/services/worker/bundle/bundle.js ./
CMD ["node", "bundle.js"]

FROM rt-slim AS events-proc
COPY --from=builder /app/services/notification/pod-events-processor/bundle/bundle.js ./
CMD ["node", "bundle.js"]

FROM rt-slim AS rating
COPY --from=builder /app/services/rating/bundle/bundle.js \
                    /app/services/rating/bundle/bundle.js.map \
                    /app/services/rating/bundle/model.json ./
EXPOSE 4700
CMD ["node", "--expose-gc", "bundle.js"]

# Optional services — uncomment target in docker-compose.coolify.yml to enable

FROM rt-slim AS sign
WORKDIR /app
COPY --from=builder /app/services/sign/pod-sign/bundle/bundle.js \
                    /app/services/sign/pod-sign/bundle/bundle.js.map ./
CMD ["dumb-init", "node", "bundle.js"]

FROM print-base AS print
WORKDIR /usr/src/app
COPY --from=builder /app/services/print/pod-print/bundle/bundle.js ./
CMD ["dumb-init", "node", "bundle.js"]
