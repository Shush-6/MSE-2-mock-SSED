# syntax=docker/dockerfile:1

# Pinned slim (Alpine) base image -> small, reproducible
FROM node:20.18.1-alpine3.20

ENV NODE_ENV=production
WORKDIR /app

# 1) Dependencies first (cached until package.json changes)
COPY package*.json ./
# 2) Production dependencies only
RUN npm install --omit=dev --no-audit --no-fund && npm cache clean --force

# 3) Application source last (changes most often)
COPY --chown=node:node src ./src

# Run as the built-in unprivileged user instead of root
USER node

EXPOSE 3000

# Used by the API service. The worker service disables it in docker-compose.yml.
HEALTHCHECK --interval=15s --timeout=5s --start-period=40s --retries=5 \
  CMD wget -qO- http://127.0.0.1:3000/api/health || exit 1

# npm start = "node src/migrate.js && node src/server.js"
# (the worker overrides this with: node src/worker.js)
CMD ["npm", "start"]
