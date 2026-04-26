# ARGs used in FROM instructions must be declared before the first FROM (global scope)
# BUILD_BASE: oven/bun:1-alpine for amd64/arm64, node:20-alpine for armv7 (Bun has no arm32 image)
# RUNTIME_BASE: node:22-alpine for amd64/arm64, node:20-alpine for armv7 (Node 22 dropped arm/v7)
ARG BUILD_BASE=oven/bun:1-alpine
ARG RUNTIME_BASE=node:22-alpine

FROM ${BUILD_BASE} AS base

# Dependencies stage
FROM base AS deps
RUN apk add --no-cache libc6-compat
WORKDIR /app

COPY package.json bun.lock* package-lock.json* ./
COPY prisma ./prisma/
RUN if command -v bun >/dev/null 2>&1; then \
      bun install --frozen-lockfile; \
    else \
      npm install; \
    fi
RUN if command -v bun >/dev/null 2>&1; then \
      bunx prisma generate --no-hints; \
    else \
      npx prisma generate --no-hints; \
    fi

# Production dependencies stage
FROM base AS prod-deps
WORKDIR /app
COPY package.json bun.lock* package-lock.json* ./
COPY prisma ./prisma/
RUN if command -v bun >/dev/null 2>&1; then \
      bun install --production --frozen-lockfile; \
    else \
      npm install --omit=dev; \
    fi
RUN if command -v bun >/dev/null 2>&1; then \
      bunx prisma generate --no-hints; \
    else \
      npx prisma generate --no-hints; \
    fi

# Builder stage
FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .

RUN if command -v bun >/dev/null 2>&1; then \
      bun run build; \
    else \
      npm run build; \
    fi

FROM ${RUNTIME_BASE} AS runner
WORKDIR /app

ENV NODE_ENV=production
ENV PORT=8000

# Install Docker CLI, Git, and procps (for accurate ps/top)
RUN apk add --no-cache docker-cli docker-cli-compose git procps bash util-linux
RUN npm install -g npm@latest

# Create non-root user
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 docklift

# Create necessary directories
RUN mkdir -p /app/data /deployments /nginx-conf
RUN chown -R docklift:nodejs /app/data /deployments /nginx-conf

# Copy built files
COPY --from=prod-deps --chown=docklift:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=docklift:nodejs /app/dist ./dist
COPY --from=builder --chown=docklift:nodejs /app/prisma ./prisma
COPY --from=builder --chown=docklift:nodejs /app/package.json ./

# Note: Running as root is required for Docker socket access
# USER docklift

EXPOSE 8000

ENV HOSTNAME="0.0.0.0"
ENV DATABASE_URL="file:/app/data/docklift.db"

# Database auto-creation: prisma db push creates the SQLite file if it doesn't exist
CMD ["sh", "-c", "npx prisma db push --skip-generate && node dist/index.js"]
