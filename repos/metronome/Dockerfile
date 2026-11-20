# Build stage: compile the Vite app into static files (dist/)
FROM node:24-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
# --base=/ serves the app from the subdomain root (e.g. metronome.pompui.de);
# vite.config.ts defaults to '/metronome/' for GitHub Pages, the CLI flag wins.
RUN npm run build -- --base=/

# Serve stage: house style — nginx-unprivileged, non-root, internal port 8080
FROM nginxinc/nginx-unprivileged:alpine
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist/ /usr/share/nginx/html/
EXPOSE 8080
