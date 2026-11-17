
# Stage 1: Build React application
FROM node:20-alpine AS build

WORKDIR /app

# Copy dependency manifests first for better layer caching
COPY package*.json ./

RUN npm install

# Copy application source and build
COPY . .

RUN npm run build

# Stage 2: Serve static files using unprivileged Nginx
FROM nginxinc/nginx-unprivileged:alpine

COPY --from=build /app/dist/ /usr/share/nginx/html/

EXPOSE 8080

# Check whether frontend is responding
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD wget -q --spider http://127.0.0.1:8080/ || exit 1
