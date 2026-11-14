# Dockerfiles that stay small and rebuild fast

## order layers by change frequency

    COPY package.json package-lock.json ./
    RUN npm ci --omit=dev
    COPY . .

Dependencies change monthly, source changes hourly. Copying source first throws away the
dependency layer on every edit.

## multi-stage, always

    FROM node:22-alpine AS build
    WORKDIR /app
    COPY package*.json ./
    RUN npm ci
    COPY . .
    RUN npm run build

    FROM node:22-alpine
    WORKDIR /app
    ENV NODE_ENV=production
    COPY --from=build /app/dist ./dist
    COPY --from=build /app/node_modules ./node_modules
    USER node
    CMD ["node", "dist/main.js"]

The final image has no compiler, no dev dependencies, no source.

## the file people forget

    # .dockerignore
    .git
    node_modules
    dist
    *.log
    .env*

Without it, COPY . . sends node_modules to the daemon and overwrites the layer you just built.

## cache mounts for package managers

    RUN --mount=type=cache,target=/root/.npm npm ci

Rebuilds reuse the registry cache without baking it into a layer.

## signals

CMD in exec form makes node PID 1, so it receives SIGTERM. Shell form puts /bin/sh as PID 1,
which does not forward the signal, and docker stop waits the full timeout before killing.

## when an image is unexpectedly large

    docker history --no-trunc image:tag | sort -k2 -h | tail
    docker run --rm -it image:tag du -sh /* 2>/dev/null | sort -h | tail
