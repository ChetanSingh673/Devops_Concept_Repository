# Devops_Concept



# Docker Multi-stage Structure

**Normal structure**
```bash
Dockerfile
│
├── 𝗦𝘁𝗮𝗴𝗲 𝟭: 𝗕𝗨𝗜𝗟𝗗
│   │
│   ├── FROM
│   ├── WORKDIR
│   ├── COPY
│   ├── Install
│   ├── COPY
│   └── Build
│
└── 𝗦𝘁𝗮𝗴𝗲 𝟮: 𝗣𝗥𝗢𝗗𝗨𝗖𝗧𝗜𝗢𝗡
    │
    ├── FROM
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
**Single Dockerfile example**
```bash
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
CMD ["node", "src/index.js"]
```
```bash
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
```


**Detailed Structure**

```bash
Dockerfile
│
├── 𝗦𝘁𝗮𝗴𝗲 𝟭: 𝗕𝗨𝗜𝗟𝗗
│   │
│   ├── FROM
│   │     └── Base image according to the application
│   │
│   ├── WORKDIR
│   │     └── Set working directory, e.g. /app
│   │
│   ├── COPY
│   │     └── Copy dependency files
│   │         ├── package.json
│   │         └── package-lock.json
│   │
│   ├── INSTALL DEPENDENCIES (RUN npm ci)
│   │     └── Install dependencies by reading dependency files like file package.json + package-lock.json
│   │         ├── Node.js → npm ci / npm install
│   │         ├── Java    → mvn dependency...
│   │         ├── Python  → pip install...
│   │         └── Go      → go mod download
│   │
│   ├── COPY
│   │     └── Copy application all files src etc...
│   │
│   └── BUILD (if required) --> This command builds your Node.js application inside the Docker image. Below you can see structure.
│         │
│         ├── npm run build
│         │       ↓
│         ├── npm checks package.json
│         │       ↓
│         ├── Finds "build": "vite build"
│         │       ↓
│         ├── Executes vite build
│         │       ↓
│         ├── Application is compiled/bundled
│         │       ↓
│         └── Build artifacts are created
│
│         Example:
│
│         /app
│         ├── src/
│         ├── public/
│         ├── package.json
│         ├── package-lock.json
│         └── dist/
│             ├── index.html
│             └── assets/
│                 ├── index-abc123.js
│                 └── index-def456.css
│
│         Note:
│         dist/ contains the production-ready frontend build artifacts.
│
│
└── 𝗦𝘁𝗮𝗴𝗲 𝟮: 𝗣𝗥𝗢𝗗𝗨𝗖𝗧𝗜𝗢𝗡
    │
    ├── FROM
    │     └── Smaller runtime/base image
    │
    ├── Create non-root user
    │
    ├── Install required runtime packages
    │
    ├── WORKDIR
    │
    ├── COPY --from=build
    │     └── Copy only required build artifacts
    │
    ├── COPY application/runtime files
    │     └── Only if required
    │
    ├── Change ownership
    │     └── chown
    │
    ├── USER
    │     └── Run container as non-root user
    │
    ├── EXPOSE
    │     └── Document application port
    │
    ├── ENTRYPOINT
    │     └── Optional fixed executable
    │
    └── CMD
          └── Default command/arguments
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
