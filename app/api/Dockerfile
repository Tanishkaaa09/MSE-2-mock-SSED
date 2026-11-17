
# Use a pinned Node.js Alpine major-version image
FROM node:20-alpine

WORKDIR /app

# Copy dependency files first for Docker layer caching
COPY --chown=node:node package*.json ./

# Install production dependencies only
RUN npm install --omit=dev

# Copy application files with non-root ownership
COPY --chown=node:node . .

# API port
EXPOSE 3000

# Run as non-root user
USER node

# Check API health endpoint
HEALTHCHECK --interval=30s --timeout=5s --start-period=30s --retries=3 \
  CMD node -e "fetch('http://127.0.0.1:3000/api/health').then(r => process.exit(r.ok ? 0 : 1)).catch(() => process.exit(1))"

# Run migrations and start API
CMD ["npm", "start"]
