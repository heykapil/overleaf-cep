# Overleaf CEP (Community Edition) - ARM64 Docker Images 

[![Docker Pulls](https://img.shields.io/docker/pulls/heykapil/overleaf?style=flat-square)](https://hub.docker.com/r/heykapil/overleaf)
[![Build Status](https://img.shields.io/github/actions/workflow/status/heykapil/overleaf-cep/main.yml?branch=docker&label=Build&style=flat-square)](https://github.com/heykapil/overleaf-cep/actions)
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

To run this on your ARM64 server, use the following `docker-compose.yml`.

**Note:** This setup requires MongoDB and Redis.

```yaml
version: '3.8'

services:
  sharelatex:
    image: heykapil/overleaf:latest
    container_name: sharelatex
    restart: always
    ports:
      - "80:80"
    volumes:
      - ./overleaf_data:/var/lib/sharelatex
    environment:
      - SHARELATEX_APP_NAME=Overleaf Community Edition
      - SHARELATEX_MONGO_URL=mongodb://mongo:27017/sharelatex
      - SHARELATEX_REDIS_HOST=redis
      - REDIS_HOST=redis
      - ENABLED_LINKED_SERVICES=true
      - ENABLED_V2_TEMPLATES=true
      # Add other standard Overleaf environment variables here
    depends_on:
      - mongo
      - redis

  mongo:
    image: mongo:5.0
    container_name: mongo
    restart: always
    volumes:
      - ./mongo_data:/data/db

  redis:
    image: redis:6.2
    container_name: redis
    restart: always
    volumes:
      - ./redis_data:/data
````

### Running the stack

```bash
docker-compose up -d
```

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

This Docker build repository is open source. Please refer to the upstream projects for the specific licensing of the Overleaf application code (AGPL-3.0).
