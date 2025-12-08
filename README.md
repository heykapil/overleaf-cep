# Overleaf CEP (Community Edition) - ARM64 Docker Images 

[![Docker Pulls](https://img.shields.io/docker/pulls/heykapil/overleaf?style=flat-square)](https://hub.docker.com/r/heykapil/overleaf)
[![Build Status](https://img.shields.io/github/actions/workflow/status/heykapil/overleaf-cep/main.yml?branch=arm64-docker&label=Build&style=flat-square)](https://github.com/heykapil/overleaf-cep/actions)
[![Platform](https://img.shields.io/badge/Platform-linux%2Farm64-orange?style=flat-square)](https://hub.docker.com/r/heykapil/overleaf/tags)

This repository provides automated, daily builds of **Overleaf Community Edition (Extended)** specifically optimized for **ARM64** architectures (Apple Silicon, Raspberry Pi, Oracle Cloud ARM, AWS Graviton).

It tracks the `ext-ce` branch of the upstream [yu-i-i/overleaf-cep](https://github.com/yu-i-i/overleaf-cep) repository.

## 🐳 Docker Image

The images are pushed to Docker Hub under:

**`heykapil/overleaf`**

Tags available:
* `latest`: The most recent successful build.
* `X.X.X`: Specific version tags matching the upstream release tags.

## ✨ Features

* **ARM64 Native:** Built specifically on `ubuntu-24.04-arm` runners for native performance on ARM devices.
* **Automatic Updates:** A cron job runs daily (`0 0 * * *`) to check for new tags in the upstream repository.
* **Smart Builds:** The workflow checks if the specific version tag already exists in the Docker registry to prevent redundant builds.
* **Custom Base:** Automatically patches the build to utilize an ARM-compatible base image (`heykapil/overleaf-base`).

## 🚀 Usage (Docker Compose)

To run this on your ARM64 server, use the following `docker-compose.yml`. A better way is to use dokploy or coolify to do this for you.

**Note:** This setup requires MongoDB and Redis. In my case, i am using dokploy, `dokploy-network` is used to expose the redis and mongo to the same network.

```yaml
services:
  mongo:
    image: mongo:6.0
    container_name: mongo
    command: "--replSet overleaf"
    restart: always
    extra_hosts:
      - "mongo:127.0.0.1"
    environment:
      MONGO_INITDB_DATABASE: sharelatex
    healthcheck:
      test: echo 'db.stats().ok' | mongosh localhost:27017/test --quiet
      interval: 10s
      timeout: 10s
      retries: 5
    volumes:
      - mongo_data:/data/db
      - /opt/overleaf-config/mongodb-init.js:/docker-entrypoint-initdb.d/mongodb-init.js
    networks:
      - dokploy-network

  redis:
    image: redis:6.2
    container_name: redis
    restart: always
    volumes:
      - redis_data:/data
    networks:
      - dokploy-network

  sharelatex:
    # STEP 1: Use your custom built image
    image: heykapil/overleaf:latest 
    # container_name: sharelatex
    restart: always
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:80/login"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 40s
    depends_on:
      - mongo
      - redis
    volumes:
      - sharelatex_data:/var/lib/overleaf
      # Optional: Mount local configs if you want to edit them without rebuilding
      # - ./overleaf_data/settings.js:/etc/overleaf/settings.js 
    environment:
      OVERLEAF_MONGO_URL: mongodb://mongo/sharelatex
      OVERLEAF_REDIS_HOST: redis
      REDIS_HOST: redis
      OVERLEAF_APP_NAME: "LaTeX"
      OVERLEAF_SITE_URL: "https://latex.example.com"
      OVERLEAF_NAV_TITLE: "LaTeX"
      OVERLEAF_HEADER_IMAGE_URL: "https://example.com/logo.png"
      NAV_HIDE_POWERED_BY: "true"
      OVERLEAF_ADMIN_EMAIL: "example@gmail.com" 
      OVERLEAF_ENABLE_DOC_HISTORY: "true"
      OVERLEAF_ENABLE_TRACK_CHANGES: "true"
      # OVERLEAF_TEMPLATE_GALLERY: "true"
      ENABLE_CONVERSIONS: "true"
      ENABLED_LINKED_FILE_TYPES: "project_file,project_output_file,url"
      EMAIL_CONFIRMATION_DISABLED: "false" 
      # OVERLEAF_EMAIL_FROM_ADDRESS: "example@gmail.com" 
      # OVERLEAF_EMAIL_SMTP_HOST: "smtp.gmail.com"
      # OVERLEAF_EMAIL_SMTP_PORT: "587"
      # OVERLEAF_EMAIL_SMTP_SECURE: "false" 
      # OVERLEAF_EMAIL_SMTP_USER: ""
      # OVERLEAF_EMAIL_SMTP_PASS: ""
      OVERLEAF_EMAIL_SMTP_TLS_REJECT_UNAUTHORIZED: "false"
      OVERLEAF_ALLOW_PUBLIC_ACCESS: "true" 
      NODE_ENV: "production"

volumes:
  mongo_data:
  redis_data:
  sharelatex_data:

networks:
  dokploy-network:
    external: true
````

### Running the stack

Create a directory `/opt/overleaf-config` for `mongodb-init.js` file. 

```
sudo mkdir -p /opt/overleaf-config
```

Create the initialization script: Copy and paste this entire block into your terminal:

```bash
cat << 'EOF' | sudo tee /opt/overleaf-config/mongodb-init.js
rs.initiate({
  _id: "overleaf",
  members: [{ _id: 0, host: "mongo:27017" }]
})
EOF
```

Deploy and start the containers.

```bash
docker-compose up -d
```

**Note:**  
1. Fix MongoDB Crash Loop (Replica Set Issue) If the Mongo container keeps restarting or ShareLaTeX cannot connect, the replica set might not have initialized. This often happens if the data volume wasn't empty on the first boot. Force initialization manually:
```bash
docker exec -it $(docker ps -qf "name=mongo") mongosh --eval "rs.initiate({ _id: 'overleaf', members: [{ _id: 0, host: 'mongo:27017' }] })"
```
Success indicator: If it prints `{ "ok" : 1 }`, the database is fixed. ShareLaTeX will connect automatically within 60 seconds.

2. Fix Blank Screen (Cloudflare Users) If you see a blank screen or infinite loading, check your browser console. If you see errors related to rocket-loader.min.js violating Content Security Policy (CSP), Cloudflare is breaking the app. Turn off the Rocket Loader in Speed > Optimization of your Cloudflare dashboard and purge cache.

## ⚙️ How the Build Works

This repository uses a GitHub Action to automate the release process:

1.  **Check Upstream:** The workflow fetches tags from `yu-i-i/overleaf-cep`.
2.  **Version Detection:** It identifies the highest version number available.
3.  **Registry Check:** It queries Docker Hub to see if an image with that version tag already exists.
4.  **Conditional Build:**
      * If the tag exists: The build is skipped to save resources.
      * If the tag is missing: The workflow checks out the specific tag code.
5.  **Patching:** The `Dockerfile` is modified on the fly to replace the standard x86 base image with `heykapil/overleaf-base:latest` (ARM64 compatible).
6.  **Push:** The image is built using `docker buildx` for the `linux/arm64` platform and pushed with both the version tag and the `latest` tag.

## 🔗 Upstream & Credits

  * **Upstream Project:** [yu-i-i/overleaf-cep](https://github.com/yu-i-i/overleaf-cep) (Extended Community Edition)
  * **Original Project:** [Overleaf/ShareLaTeX](https://github.com/overleaf/overleaf)

## 📄 License

This Docker build repository is open source. Please refer to the upstream projects for the specific licensing of the Overleaf application code (AGPL-3.0). The docker images provided are 'as is' without warranty of any kind and for research and educational purpose. 
