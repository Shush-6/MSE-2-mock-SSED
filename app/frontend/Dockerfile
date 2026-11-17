# syntax=docker/dockerfile:1

# ---------- Stage 1: build the React app with Node ----------
FROM node:20.18.1-alpine3.20 AS build
WORKDIR /app

# Copy only the dependency manifests first so this layer is cached
# until package.json changes.
COPY package*.json ./
RUN npm install --no-audit --no-fund

# Now copy the source and build -> output goes to /app/dist
COPY . .
RUN npm run build

# ---------- Stage 2: serve static files with unprivileged Nginx ----------
# No Node.js, no node_modules in this stage -> small and safer image.
FROM nginxinc/nginx-unprivileged:1.27-alpine
COPY --from=build /app/dist /usr/share/nginx/html

# The base image already runs as the non-root user "nginx" (uid 101).
# We state it explicitly so it is obvious (and checkable) in the Dockerfile.
USER nginx

EXPOSE 8080

HEALTHCHECK --interval=15s --timeout=5s --start-period=10s --retries=5 \
  CMD wget -q --spider http://127.0.0.1:8080/ || exit 1
