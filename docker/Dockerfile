###########################
#     BASE CONTAINER      #
###########################
FROM node:18-alpine AS base

###########################
#    BUILDER CONTAINER    #
###########################
FROM base AS builder

# Check https://github.com/nodejs/docker-node/tree/b4117f9333da4138b03a546ec926ef50a31506c3#nodealpine to understand why libc6-compat might be needed.
RUN apk add --no-cache libc6-compat
RUN apk add --no-cache jq
WORKDIR /app

COPY . .

RUN npm install -g "turbo@^1.9.3"

# Outputs to the /out folder
# source: https://turbo.build/repo/docs/reference/command-line-reference/prune#--docker
RUN turbo prune --scope=@documenso/web --docker

###########################
#   INSTALLER CONTAINER   #
###########################
FROM base AS installer

# Check https://github.com/nodejs/docker-node/tree/b4117f9333da4138b03a546ec926ef50a31506c3#nodealpine to understand why libc6-compat might be needed.
RUN apk add --no-cache libc6-compat
RUN apk add --no-cache jq
# Required for node_modules/aws-crt
RUN apk add --no-cache make cmake g++

WORKDIR /app

# Disable husky from installing hooks
ENV HUSKY 0
ENV DOCKER_OUTPUT 1
ENV NEXT_TELEMETRY_DISABLED 1

# Encryption keys
ARG NEXT_PRIVATE_ENCRYPTION_KEY="CAFEBABE"
ENV NEXT_PRIVATE_ENCRYPTION_KEY="$NEXT_PRIVATE_ENCRYPTION_KEY"

ARG NEXT_PRIVATE_ENCRYPTION_SECONDARY_KEY="DEADBEEF"
ENV NEXT_PRIVATE_ENCRYPTION_SECONDARY_KEY="$NEXT_PRIVATE_ENCRYPTION_SECONDARY_KEY"


# Uncomment and use build args to enable remote caching
# ARG TURBO_TEAM
# ENV TURBO_TEAM=$TURBO_TEAM
# ARG TURBO_TOKEN
# ENV TURBO_TOKEN=$TURBO_TOKEN

# Env variables
ARG NEXTAUTH_URL="http://localhost:3000"
ENV NEXTAUTH_URL="$NEXTAUTH_URL"

ARG NEXTAUTH_SECRET="secret"
ENV NEXTAUTH_SECRET="$NEXTAUTH_SECRET"

ARG NEXT_PUBLIC_WEBAPP_URL="http://localhost:3000"
ENV NEXT_PUBLIC_WEBAPP_URL="$NEXT_PUBLIC_WEBAPP_URL"

ARG NEXT_PRIVATE_INTERNAL_WEBAPP_URL="http://localhost:3000"
ENV NEXT_PRIVATE_INTERNAL_WEBAPP_URL="$NEXT_PRIVATE_INTERNAL_WEBAPP_URL"

ARG NEXT_PRIVATE_DATABASE_URL="http://localhost:3000"
ENV NEXT_PRIVATE_DATABASE_URL="$NEXT_PRIVATE_DATABASE_URL"

ARG NEXT_PRIVATE_DIRECT_DATABASE_URL="mysql://root:@127.0.0.1:3306/gmc_cars?serverVersion=8.0.32&charset=utf8mb4"
ENV NEXT_PRIVATE_DIRECT_DATABASE_URL="$NEXT_PRIVATE_DIRECT_DATABASE_URL"

ARG NEXT_PRIVATE_SMTP_TRANSPORT="smtp-auth"
ENV NEXT_PRIVATE_SMTP_TRANSPORT="$NEXT_PRIVATE_SMTP_TRANSPORT"

ARG NEXT_PRIVATE_SMTP_FROM_NAME="<your-next-private-smtp-from-name>"
ENV NEXT_PRIVATE_SMTP_FROM_NAME="<your-next-private-smtp-from-name>"

ARG NEXT_PRIVATE_SMTP_HOST="sandbox.smtp.mailtrap.io"
ENV NEXT_PRIVATE_SMTP_HOST="$NEXT_PRIVATE_SMTP_HOST"

ARG NEXT_PRIVATE_SMTP_PORT="587"
ENV NEXT_PRIVATE_SMTP_PORT="$NEXT_PRIVATE_SMTP_PORT"

ARG NEXT_PRIVATE_SMTP_FROM_NAME="GMC Car's"
ENV NEXT_PRIVATE_SMTP_FROM_NAME="$NEXT_PRIVATE_SMTP_FROM_NAME"

ARG NEXT_PRIVATE_SMTP_FROM_ADDRESS="nepasrepondre@gmccars.fr"
ENV NEXT_PRIVATE_SMTP_FROM_ADDRESS="$NEXT_PRIVATE_SMTP_FROM_ADDRESS"

# First install the dependencies (as they change less often)
COPY .gitignore .gitignore
COPY --from=builder /app/out/json/ .
COPY --from=builder /app/out/package-lock.json ./package-lock.json

RUN npm ci

# Then copy all the source code (as it changes more often)
COPY --from=builder /app/out/full/ .
# Finally copy the turbo.json file so that we can run turbo commands
COPY turbo.json turbo.json

RUN npm install -g "turbo@^1.9.3"

RUN turbo run build --filter=@documenso/web...

###########################
#     RUNNER CONTAINER    #
###########################
FROM base AS runner

WORKDIR /app

# Don't run production as root
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

USER nextjs

COPY --from=installer /app/apps/web/next.config.js .
COPY --from=installer /app/apps/web/package.json .

# Automatically leverage output traces to reduce image size
# https://nextjs.org/docs/advanced-features/output-file-tracing
COPY --from=installer --chown=nextjs:nodejs /app/apps/web/.next/standalone ./
COPY --from=installer --chown=nextjs:nodejs /app/apps/web/.next/static ./apps/web/.next/static
COPY --from=installer --chown=nextjs:nodejs /app/apps/web/public ./apps/web/public

# Copy the prisma binary, schema and migrations
COPY --from=installer --chown=nextjs:nodejs /app/packages/prisma/schema.prisma ./packages/prisma/schema.prisma
COPY --from=installer --chown=nextjs:nodejs /app/packages/prisma/migrations ./packages/prisma/migrations
COPY --from=installer --chown=nextjs:nodejs /app/node_modules/prisma/ ./node_modules/prisma/
COPY --from=installer --chown=nextjs:nodejs /app/node_modules/@prisma/ ./node_modules/@prisma/

# Symlink the prisma binary
RUN mkdir node_modules/.bin
RUN ln -s /app/node_modules/prisma/build/index.js ./node_modules/.bin/prisma


# Get the start script from docker/start.sh
COPY --chown=nextjs:nodejs ./docker/start.sh ./start.sh

CMD ["sh", "start.sh"]
