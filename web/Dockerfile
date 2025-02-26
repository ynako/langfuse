# ---------- base 이미지 ----------
    FROM --platform=${TARGETPLATFORM:-linux/amd64} node:20-alpine3.20 AS base

    # Alpine 기본 업데이트 & 네이티브 빌드 도구 설치 (python3, make, g++)
    RUN apk update && apk upgrade --no-cache \
        && apk add --no-cache libcrypto3 libssl3 libc6-compat busybox ssl_client \
        && apk add --no-cache python3 make g++
    
    # Turbo & PNPM 설정
    RUN npm install turbo@^1.13.4 --global
    ENV PNPM_HOME="/pnpm"
    ENV PATH="$PNPM_HOME:$PATH"
    RUN npm install -g corepack
    RUN corepack enable
    RUN corepack prepare pnpm@9.5.0 --activate
    
    # ---------- builder ----------
    FROM base AS builder
    WORKDIR /app
    
    # (1) package.json & pnpm-lock.yaml 먼저 복사 → install
    COPY package.json pnpm-lock.yaml ./
    RUN pnpm install
    
    # (2) 나머지 모든 소스 복사
    COPY . .
    
    # (3) 필요 없는 파일 제거 (middleware.ts 등)
    RUN rm -f ./web/src/middleware.ts
    
    # 환경 변수 & 빌드 설정
    ENV NEXT_TELEMETRY_DISABLED=1
    ENV NEXT_MANUAL_SIG_HANDLE=true
    ENV CI=true
    ENV NODE_OPTIONS='--max-old-space-size=2048'
    
    # (4) 빌드 (web만 빌드)
    RUN pnpm run build --filter=web...
    
    # ---------- runner ----------
    FROM base AS runner
    WORKDIR /app
    
    ENV NODE_ENV=production
    ENV NEXT_TELEMETRY_DISABLED=1
    ENV DOCKER_BUILD=0
    ENV NEXT_MANUAL_SIG_HANDLE=true
    
    # dumb-init, tzdata 설치
    RUN apk add --no-cache dumb-init tzdata
    
    
    # (1) 앱 실행에 필요한 파일들만 복사
    COPY --from=builder /app/web/next.config.mjs .
    COPY --from=builder /app/web/package.json .
    COPY --from=builder /app/web/.next/standalone ./
    COPY --from=builder /app/web/.next/static ./web/.next/static
    COPY --from=builder /app/web/public ./web/public
    
    COPY --from=builder /app/packages/shared/prisma ./packages/shared/prisma
    COPY --from=builder /app/packages/shared/clickhouse ./packages/shared/clickhouse
    
    # (2) entrypoint & cleanup script
    COPY --chown=node:node ./web/entrypoint.sh ./web/entrypoint.sh
    COPY --chown=node:node ./packages/shared/scripts/cleanup.sql ./packages/shared/scripts/cleanup.sql
    RUN chmod +x ./web/entrypoint.sh
    
    # (3) 비root 사용자 생성
    ARG UID=1001
    ARG GID=1001
    RUN addgroup --system --gid ${GID} nodejs \
        && adduser --system --uid ${UID} nodejs
    USER nodejs
    
    # (4) 포트 & 실행 명령
    ENV PORT=3000
    EXPOSE 3000
    ENTRYPOINT ["dumb-init", "--", "./web/entrypoint.sh"]
    CMD ["node", "./web/server.js", "--keepAliveTimeout", "110000"]
    