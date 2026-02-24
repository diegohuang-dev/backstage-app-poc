# Stage 1 - Build the backend
FROM node:22-bookworm-slim AS build

# Install dependencies for building native modules
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked     --mount=type=cache,target=/var/lib/apt,sharing=locked     apt-get update &&     apt-get install -y --no-install-recommends python3 g++ build-essential libsqlite3-dev &&     rm -rf /var/lib/apt/lists/*

USER node
WORKDIR /app

# Copy the yarn setup
COPY --chown=node:node .yarn ./.yarn
COPY --chown=node:node .yarnrc.yml ./

# Copy package.json and lockfile
COPY --chown=node:node package.json yarn.lock backstage.json ./

# Copy all packages (we need them for workspaces to resolve)
COPY --chown=node:node packages ./packages
# Copy plugins if any
COPY --chown=node:node plugins ./plugins
# Copy examples if any
COPY --chown=node:node examples ./examples

# Install dependencies
RUN --mount=type=cache,target=/home/node/.cache/yarn,sharing=locked,uid=1000,gid=1000     yarn install --immutable

# Build the backend
# This will generate packages/backend/dist/skeleton.tar.gz and packages/backend/dist/bundle.tar.gz
RUN yarn tsc
RUN yarn --cwd packages/backend build

# Stage 2 - Create the production image
FROM node:22-bookworm-slim

# Install runtime dependencies (sqlite3 if needed, python for some native modules if rebuilt)
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked     --mount=type=cache,target=/var/lib/apt,sharing=locked     apt-get update &&     apt-get install -y --no-install-recommends libsqlite3-dev python3 build-essential &&     rm -rf /var/lib/apt/lists/*

USER node
WORKDIR /app

# Copy yarn setup
COPY --chown=node:node .yarn ./.yarn
COPY --chown=node:node .yarnrc.yml ./
COPY --chown=node:node backstage.json ./

ENV NODE_ENV=production
ENV NODE_OPTIONS="--no-node-snapshot"

# Copy skeleton from build stage
COPY --from=build --chown=node:node /app/packages/backend/dist/skeleton.tar.gz ./
RUN tar xzf skeleton.tar.gz && rm skeleton.tar.gz

# Install production dependencies
RUN --mount=type=cache,target=/home/node/.cache/yarn,sharing=locked,uid=1000,gid=1000     yarn workspaces focus --all --production && rm -rf "$(yarn cache clean)"

# Copy bundle from build stage
COPY --from=build --chown=node:node /app/packages/backend/dist/bundle.tar.gz ./
RUN tar xzf bundle.tar.gz && rm bundle.tar.gz

# Copy config files
COPY --chown=node:node app-config*.yaml ./

CMD ["node", "packages/backend", "--config", "app-config.yaml", "--config", "app-config.production.yaml"]
