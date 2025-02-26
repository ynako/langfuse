FROM --platform=${TARGETPLATFORM:-linux/amd64} node:20-alpine3.20 AS base
RUN apk update && apk upgrade --no-cache libcrypto3 libssl3 libc6-compat busybox ssl_client
RUN npm install turbo@^1.13.4 --global
ENV PNPM_HOME="/pnpm"
ENV PATH="$PNPM_HOME:$PATH"
RUN npm install -g corepack
RUN corepack enable
RUN corepack prepare pnpm@9.5.0 --activate

FROM base AS builder
WORKDIR /app

# 1) copy minimal files for install
COPY package.json pnpm-lock.yaml ./

# 2) install deps
RUN pnpm install --frozen-lockfile

# 3) copy the rest
COPY . .

# remove middleware.ts if it exists
RUN rm -f ./web/src/middleware.ts

# Next.js telemetry off
ENV NEXT_TELEMETRY_DISABLED=1
ENV NEXT_MANUAL_SIG_HANDLE=true
ENV CI=true
ENV NODE_OPTIONS='--max-old-space-size=2048'

# build
RUN pnpm run build --filter=web...

# 4) runner stage
FROM base AS runner
WORKDIR /app
ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1
ENV DOCKER_BUILD=0
ENV NEXT_MANUAL_SIG_HANDLE=true
RUN apk add --no-cache dumb-init tzdata

# create non-root user
ARG UID=1001
ARG GID=1001
RUN addgroup --system --gid ${GID} nodejs
RUN adduser  --system --uid ${UID} nextjs

# install prisma for migrations
RUN npm install -g --no-package-lock --no-save prisma@6.3.0

# copy build results
COPY --from=builder /app/web/next.config.mjs . 
COPY --from=builder /app/web/package.json .
COPY --from=builder /app/web/.next/standalone ./
COPY --from=builder /app/web/.next/static ./web/.next/static
COPY --from=builder /app/web/public ./web/public
COPY --from=builder /app/packages/shared/prisma ./packages/shared/prisma
COPY --from=builder /app/packages/shared/clickhouse ./packages/shared/clickhouse
COPY --chown=nextjs:nodejs ./web/entrypoint.sh ./web/entrypoint.sh
COPY --chown=nextjs:nodejs ./packages/shared/scripts/cleanup.sql ./packages/shared/scripts/cleanup.sql
RUN chmod +x ./web/entrypoint.sh

USER nextjs
ENV PORT=3000
ENTRYPOINT ["dumb-init", "--", "./web/entrypoint.sh"]
CMD ["node", "./web/server.js", "--keepAliveTimeout", "110000"]
