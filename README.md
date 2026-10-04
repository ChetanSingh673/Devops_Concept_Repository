# Devops_Concept_Repository

# Docker Multi-stage Structure

```bash
Dockerfile
│
├── Stage 1: BUILD
│   │
│   ├── FROM    (Base image defined according to the source code)
│   ├── WORKDIR (/app)
│   ├── COPY    (Dependency ---> package.json)
│   ├── Install (Go to /app directory and install the dependency by reading the file (package.json + package-lock.json) using the npm,mvn,go,pip)
|   |── COPY    (All files)
│   └── Build   (if required) (npm look inside the package.json and finds: "build": "vite build". Then npm executes: vite build) (This command builds your Node.js application inside the Docker image. Below you can see)
│          |──── Your source code
|          |       ↓
|          |──── COPY . .
|          |       ↓ 
|          |──── npm run build
|          |       ↓
|          |──── Application is compiled/bundled
|          |       ↓
|          └──── Build artifact is created in /app folder
|                 You may get in the folder like below
|                     ├── src/
|                     ├── public/
|                     ├── package.json
|                     ├── package-lock.json
|                     └── dist/
|                           ├── index.html
|                           ├── assets/
|                           |      ├── index-abc123.js
|                           |      └── index-def456.css
|                           └── ...
|                    Note: That dist/ directory contains the production-ready build artifacts.                         
|
|
|
|     
│
└── Stage 2: PRODUCTION
    │
    ├── FROM node
    ├── Create non-root user
    ├── Install required runtime packages
    ├── WORKDIR
    ├── COPY --from=build
    ├── COPY application code
    ├── Change ownership
    ├── USER
    ├── EXPOSE
    ├── ENTRYPOINT
    └── CMD
 ```   
```bash
FROM <base-image> AS build

# Install dependencies
# Build application
# Generate production artifacts


FROM <base-image> AS production

# Copy only what is required
COPY --from=build <source> <destination>

# Run application
```
During the build stage, the application source code is compiled and bundled, producing build artifacts. In a multi-stage Docker build, we copy only those artifacts into the final Nginx image

**Dockerfile Example**
```bash
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
CMD ["node", "src/index.js"]
```

WORKDIR /app
      ↓
COPY package*.json ./
      ↓
package.json + package-lock.json
      ↓
RUN npm ci
      ↓
npm reads those files automatically
      ↓
dependencies installed
      ↓
node_modules/

**Frontend docker multi stage file**
```bash
#=======================================================================
#Build Stage
#=======================================================================
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci && npm cache clean --force
COPY . .
RUN npm run build 

#========================================================================
# Production Stage
#========================================================================
FROM nginx:1.27-alpine AS production

# Security: remove default nginx config
RUN rm -rf /etc/nginx/conf.d/default.conf  /usr/share/nginx/html/*

# Copy custum nginx config 
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Copy compiled frontend from build stage
COPY --from=build /app/dist /usr/share/nginx/html

# Security: run nginx as non-root
RUN chown nginx:nginx /usr/share/nginx/html && \
    chown nginx:nginx /var/cache/nginx && \
    chown nginx:nginx /var/log/nginx && \
    touch /var/run/nginx.pid && \
    chown -R nginx:nginx /var/run/nginx.pid 

USER nginx

EXPOSE 8080
CMD ["nginx", "-g", "daemon off;"]
```
