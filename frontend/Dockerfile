# ARGs used in FROM instructions must be declared before the first FROM (global scope)
# BUILD_BASE: oven/bun:1-alpine for amd64/arm64, node:20-alpine for armv7 (Bun has no arm32 image)
# RUNTIME_BASE: node:22-alpine for amd64/arm64, node:20-alpine for armv7 (Node 22 dropped arm/v7)
ARG BUILD_BASE=oven/bun:1-alpine
ARG RUNTIME_BASE=node:22-alpine

FROM ${BUILD_BASE} AS deps
RUN apk add --no-cache libc6-compat
WORKDIR /app

COPY package.json bun.lock* package-lock.json* ./
RUN if command -v bun >/dev/null 2>&1; then \
      bun install --frozen-lockfile; \
    else \
      npm install; \
    fi

# Build with Node.js (stable — Bun crashes with SIGILL on some server CPUs)
FROM ${RUNTIME_BASE} AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .

ENV NEXT_TELEMETRY_DISABLED=1

# Ensure public folder exists
RUN mkdir -p public

RUN npm run build

# Production runtime
FROM ${RUNTIME_BASE} AS runner
WORKDIR /app

ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

# Create public folder if it doesn't exist
RUN mkdir -p public

COPY --from=builder /app/public ./public

RUN mkdir .next
RUN chown nextjs:nodejs .next

COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs

EXPOSE 3000

ENV PORT=3000
ENV HOSTNAME="0.0.0.0"

CMD ["node", "server.js"]
