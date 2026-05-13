# Cross-Platform Multi-Architecture Build Guide Using Docker Buildx

This guide provides comprehensive instructions for building Self Node Remediation operator images for multiple architectures (amd64, s390x, arm64, ppc64le) from a single machine using Docker buildx.

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Setup Docker Buildx](#setup-docker-buildx)
4. [Building Multi-Arch Operator Image](#building-multi-arch-operator-image)
5. [Building Multi-Arch Bundle Image](#building-multi-arch-bundle-image)
6. [Building Multi-Arch Catalog Image](#building-multi-arch-catalog-image)
7. [Creating Multi-Arch Manifests](#creating-multi-arch-manifests)
8. [Complete Build Script](#complete-build-script)
9. [Troubleshooting](#troubleshooting)

---

## Overview

### What is Docker Buildx?

Docker Buildx is a CLI plugin that extends Docker with the ability to build multi-platform images using BuildKit. It allows you to:

- Build images for multiple architectures from a single machine
- Create multi-arch manifest lists
- Use QEMU emulation for cross-compilation
- Push images directly to registries

### Why Use Buildx for Multi-Arch Builds?

**Traditional Approach (Native Builds):**
```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  amd64      │     │   s390x     │     │   arm64     │
│  Machine    │     │   Machine   │     │   Machine   │
│             │     │             │     │             │
│  Build      │     │  Build      │     │  Build      │
│  amd64      │     │  s390x      │     │  arm64      │
│  Image      │     │  Image      │     │  Image      │
└─────────────┘     └─────────────┘     └─────────────┘
```
**Requires:** 3 different machines, manual coordination

**Buildx Approach (Cross-Platform):**
```
┌─────────────────────────────────────────────────┐
│           Single Build Machine (any arch)        │
│                                                  │
│  ┌──────────────────────────────────────────┐  │
│  │         Docker Buildx + QEMU             │  │
│  │                                          │  │
│  │  Build amd64  │  Build s390x  │  Build  │  │
│  │  Image        │  Image        │  arm64   │  │
│  │               │               │  Image   │  │
│  └──────────────────────────────────────────┘  │
│                                                  │
│  Creates Multi-Arch Manifest List               │
└─────────────────────────────────────────────────┘
```
**Requires:** 1 machine, automated process

### Architecture Support Matrix

| Architecture | Status | Notes |
|--------------|--------|-------|
| amd64 (x86_64) | ✅ Fully Supported | Primary architecture |
| s390x (IBM Z) | ✅ Fully Supported | Tested on OpenShift |
| arm64 (aarch64) | 🔄 Framework Ready | Can be enabled |
| ppc64le (POWER) | 🔄 Framework Ready | Can be enabled |

---

## Prerequisites

### 1. Install Docker with Buildx

**Check if buildx is available:**
```bash
docker buildx version
```

**If not available, install Docker Desktop or Docker CE 19.03+:**

**For Linux:**
```bash
# Install Docker CE
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Verify buildx
docker buildx version
```

**For macOS:**
```bash
# Install Docker Desktop
brew install --cask docker

# Start Docker Desktop and verify
docker buildx version
```

### 2. Install QEMU for Emulation

QEMU allows building for architectures different from your host machine.

**For Linux:**
```bash
# Install QEMU
sudo apt-get update
sudo apt-get install -y qemu-user-static binfmt-support

# Register QEMU handlers
docker run --rm --privileged multiarch/qemu-user-static --reset -p yes

# Verify QEMU is registered
ls -la /proc/sys/fs/binfmt_misc/
```

**For macOS:**
```bash
# Docker Desktop includes QEMU
# Just verify it's working
docker run --rm --privileged multiarch/qemu-user-static --reset -p yes
```

**For s390x (if building from s390x):**
```bash
# Install QEMU for cross-compilation to other architectures
sudo dnf install -y qemu-user-static
docker run --rm --privileged multiarch/qemu-user-static --reset -p yes
```

### 3. Login to Container Registry

```bash
# Login to Quay.io
docker login quay.io
# Enter your credentials

# Or use podman
podman login quay.io
```

### 4. Set Environment Variables

```bash
# REQUIRED: Set your registry
export IMAGE_REGISTRY=quay.io/YOUR_USERNAME

# Image names with version tag
export VERSION=0.8.0
export IMG=${IMAGE_REGISTRY}/self-node-remediation-operator:${VERSION}
export BUNDLE_IMG=${IMAGE_REGISTRY}/self-node-remediation-operator-bundle:${VERSION}
export CATALOG_IMG=${IMAGE_REGISTRY}/self-node-remediation-operator-catalog:${VERSION}

# Verify
echo "Registry: ${IMAGE_REGISTRY}"
echo "Operator: ${IMG}"
echo "Bundle: ${BUNDLE_IMG}"
echo "Catalog: ${CATALOG_IMG}"
```

---

## Setup Docker Buildx

### 1. Create a New Builder Instance

```bash
# Create a new builder with multi-platform support
docker buildx create \
  --name multiarch-builder \
  --driver docker-container \
  --use

# Verify builder is created
docker buildx ls
```

**Expected output:**
```
NAME/NODE            DRIVER/ENDPOINT             STATUS   PLATFORMS
multiarch-builder *  docker-container
  multiarch-builder0 unix:///var/run/docker.sock running  linux/amd64, linux/arm64, linux/s390x, linux/ppc64le
```

### 2. Bootstrap the Builder

```bash
# Start the builder container
docker buildx inspect --bootstrap

# Verify supported platforms
docker buildx inspect multiarch-builder
```

**Expected output should include:**
```
Platforms: linux/amd64, linux/arm64, linux/s390x, linux/ppc64le, ...
```

### 3. Test Multi-Arch Build

```bash
# Test with a simple image
docker buildx build \
  --platform linux/amd64,linux/s390x \
  --tag test-multiarch:latest \
  --file - . <<EOF
FROM alpine:latest
RUN uname -m > /arch.txt
CMD cat /arch.txt
EOF

# This should build successfully for both architectures
```

---

## Building Multi-Arch Operator Image

### Method 1: Build and Push for Multiple Architectures

This is the **recommended method** - builds for all architectures and pushes directly to registry.

```bash
# Build for amd64 and s390x, push to registry
docker buildx build \
  --platform linux/amd64,linux/s390x \
  --tag ${IMG} \
  --file Dockerfile \
  --push \
  .

# This will:
# 1. Build operator binary for amd64
# 2. Build operator binary for s390x
# 3. Create container images for both
# 4. Create a multi-arch manifest list
# 5. Push everything to registry
```

**Build for all supported architectures:**
```bash
# Build for amd64, s390x, arm64, and ppc64le
docker buildx build \
  --platform linux/amd64,linux/s390x,linux/arm64,linux/ppc64le \
  --tag ${IMG} \
  --file Dockerfile \
  --push \
  .
```

### Method 2: Build and Load Locally (Single Architecture)

If you want to test locally before pushing:

```bash
# Build for current architecture only and load into Docker
docker buildx build \
  --platform linux/$(uname -m | sed 's/x86_64/amd64/') \
  --tag ${IMG} \
  --file Dockerfile \
  --load \
  .

# Test the image
docker run --rm ${IMG} /manager --version
```

**Note:** `--load` only works with single platform builds. For multi-platform, use `--push`.

### Method 3: Build for Specific Architecture

```bash
# Build only for s390x
docker buildx build \
  --platform linux/s390x \
  --tag ${IMG}-s390x \
  --file Dockerfile \
  --push \
  .

# Build only for amd64
docker buildx build \
  --platform linux/amd64 \
  --tag ${IMG}-amd64 \
  --file Dockerfile \
  --push \
  .
```

### Verify Multi-Arch Operator Image

```bash
# Inspect the manifest list
docker buildx imagetools inspect ${IMG}

# Expected output:
# Name:      quay.io/YOUR_USERNAME/self-node-remediation-operator:0.8.0
# MediaType: application/vnd.docker.distribution.manifest.list.v2+json
# Digest:    sha256:...
#
# Manifests:
#   Name:      quay.io/YOUR_USERNAME/self-node-remediation-operator:0.8.0@sha256:...
#   MediaType: application/vnd.docker.distribution.manifest.v2+json
#   Platform:  linux/amd64
#
#   Name:      quay.io/YOUR_USERNAME/self-node-remediation-operator:0.8.0@sha256:...
#   MediaType: application/vnd.docker.distribution.manifest.v2+json
#   Platform:  linux/s390x
```

### Test on Different Architectures

```bash
# Pull and run on amd64
docker pull --platform linux/amd64 ${IMG}
docker run --rm --platform linux/amd64 ${IMG} /manager --version

# Pull and run on s390x
docker pull --platform linux/s390x ${IMG}
docker run --rm --platform linux/s390x ${IMG} /manager --version

# Docker automatically selects the right architecture
docker run --rm ${IMG} /manager --version
```

---

## Building Multi-Arch Bundle Image

Bundle images are typically architecture-independent (they contain YAML files), but we'll build them for consistency.

### 1. Generate Bundle Manifests

```bash
# Generate bundle
make bundle VERSION=${VERSION}

# Update CSV with correct operator image
sed -i "s|containerImage:.*|containerImage: ${IMG}|" \
  bundle/manifests/self-node-remediation.clusterserviceversion.yaml

# Update operator image references
sed -i "s|image:.*controller.*|image: ${IMG}|" \
  bundle/manifests/self-node-remediation.clusterserviceversion.yaml
```

### 2. Build Multi-Arch Bundle Image

```bash
# Build bundle for multiple architectures
docker buildx build \
  --platform linux/amd64,linux/s390x \
  --tag ${BUNDLE_IMG} \
  --file bundle.Dockerfile \
  --push \
  .
```

**Or for all architectures:**
```bash
docker buildx build \
  --platform linux/amd64,linux/s390x,linux/arm64,linux/ppc64le \
  --tag ${BUNDLE_IMG} \
  --file bundle.Dockerfile \
  --push \
  .
```

### 3. Verify Bundle Image

```bash
# Inspect bundle manifest
docker buildx imagetools inspect ${BUNDLE_IMG}

# Test bundle contents
docker run --rm ${BUNDLE_IMG} ls -la /manifests/
```

---

## Building Multi-Arch Catalog Image

### 1. Create Catalog Structure

```bash
# Create catalog directory
mkdir -p catalog

# Initialize catalog
opm init self-node-remediation \
  --default-channel=stable \
  --description=./README.md \
  --icon=./config/assets/snr_icon_blue.png \
  --output yaml \
  > catalog/index.yaml

# Render bundle into catalog
opm render ${BUNDLE_IMG} --output yaml >> catalog/index.yaml

# Add channel entry
cat <<EOF >> catalog/index.yaml
---
schema: olm.channel
package: self-node-remediation
name: stable
entries:
  - name: self-node-remediation.v${VERSION}
EOF

# Validate catalog
opm validate catalog
```

### 2. Generate Catalog Dockerfile

```bash
# Generate Dockerfile for catalog
opm generate dockerfile catalog

# This creates catalog.Dockerfile
```

### 3. Build Multi-Arch Catalog Image

```bash
# Build catalog for multiple architectures
docker buildx build \
  --platform linux/amd64,linux/s390x \
  --tag ${CATALOG_IMG} \
  --file catalog.Dockerfile \
  --push \
  .
```

**Or for all architectures:**
```bash
docker buildx build \
  --platform linux/amd64,linux/s390x,linux/arm64,linux/ppc64le \
  --tag ${CATALOG_IMG} \
  --file catalog.Dockerfile \
  --push \
  .
```

### 4. Verify Catalog Image

```bash
# Inspect catalog manifest
docker buildx imagetools inspect ${CATALOG_IMG}

# Test catalog
docker run --rm ${CATALOG_IMG} ls -la /
```

---

## Creating Multi-Arch Manifests

### Understanding Manifest Lists

A manifest list (also called a "fat manifest") is a single image reference that points to multiple architecture-specific images.

```
quay.io/user/operator:v1.0
├── linux/amd64 → sha256:abc123...
├── linux/s390x → sha256:def456...
├── linux/arm64 → sha256:ghi789...
└── linux/ppc64le → sha256:jkl012...
```

When you pull the image, Docker automatically selects the correct architecture.

### Method 1: Using Buildx (Automatic)

When you use `docker buildx build --push` with multiple platforms, buildx automatically creates the manifest list.

```bash
# This automatically creates a manifest list
docker buildx build \
  --platform linux/amd64,linux/s390x \
  --tag ${IMG} \
  --push \
  .
```

### Method 2: Manual Manifest Creation

If you built images separately, you can create a manifest list manually:

```bash
# Build images for each architecture separately
docker buildx build --platform linux/amd64 --tag ${IMG}-amd64 --push .
docker buildx build --platform linux/s390x --tag ${IMG}-s390x --push .

# Create manifest list
docker manifest create ${IMG} \
  ${IMG}-amd64 \
  ${IMG}-s390x

# Annotate each image (optional but recommended)
docker manifest annotate ${IMG} ${IMG}-amd64 --arch amd64 --os linux
docker manifest annotate ${IMG} ${IMG}-s390x --arch s390x --os linux

# Push manifest list
docker manifest push ${IMG}
```

### Method 3: Using Buildx Imagetools

```bash
# Create manifest from existing images
docker buildx imagetools create \
  --tag ${IMG} \
  ${IMG}-amd64 \
  ${IMG}-s390x
```

### Inspect Manifest Lists

```bash
# View manifest list
docker manifest inspect ${IMG}

# Or using buildx
docker buildx imagetools inspect ${IMG}

# View specific architecture
docker manifest inspect ${IMG} | jq '.manifests[] | select(.platform.architecture=="s390x")'
```

---

## Complete Build Script

Here's a complete script to build all images for multiple architectures:

```bash
#!/bin/bash
set -e

echo "=========================================="
echo "Multi-Architecture Build Script"
echo "=========================================="

# Configuration
export IMAGE_REGISTRY=${IMAGE_REGISTRY:-quay.io/YOUR_USERNAME}
export VERSION=${VERSION:-0.8.0}
export PLATFORMS=${PLATFORMS:-linux/amd64,linux/s390x}

# Image names
export IMG=${IMAGE_REGISTRY}/self-node-remediation-operator:${VERSION}
export BUNDLE_IMG=${IMAGE_REGISTRY}/self-node-remediation-operator-bundle:${VERSION}
export CATALOG_IMG=${IMAGE_REGISTRY}/self-node-remediation-operator-catalog:${VERSION}

echo "Registry: ${IMAGE_REGISTRY}"
echo "Version: ${VERSION}"
echo "Platforms: ${PLATFORMS}"
echo "Operator: ${IMG}"
echo "Bundle: ${BUNDLE_IMG}"
echo "Catalog: ${CATALOG_IMG}"
echo "=========================================="

# Verify buildx is available
if ! docker buildx version &> /dev/null; then
    echo "❌ Error: docker buildx is not available"
    exit 1
fi

# Create/use builder
echo "Setting up buildx builder..."
docker buildx create --name multiarch-builder --use --driver docker-container 2>/dev/null || \
docker buildx use multiarch-builder

# Bootstrap builder
docker buildx inspect --bootstrap

echo ""
echo "=========================================="
echo "Step 1: Building Operator Image"
echo "=========================================="
docker buildx build \
  --platform ${PLATFORMS} \
  --tag ${IMG} \
  --file Dockerfile \
  --push \
  --progress=plain \
  .

echo "✅ Operator image built and pushed"
docker buildx imagetools inspect ${IMG}

echo ""
echo "=========================================="
echo "Step 2: Generating Bundle"
echo "=========================================="
make bundle VERSION=${VERSION}

# Update CSV with correct operator image
echo "Updating CSV with operator image: ${IMG}"
sed -i.bak "s|containerImage:.*|containerImage: ${IMG}|" \
  bundle/manifests/self-node-remediation.clusterserviceversion.yaml
sed -i.bak "s|image:.*controller.*|image: ${IMG}|" \
  bundle/manifests/self-node-remediation.clusterserviceversion.yaml

# Validate bundle
make bundle-validate

echo "✅ Bundle manifests generated and validated"

echo ""
echo "=========================================="
echo "Step 3: Building Bundle Image"
echo "=========================================="
docker buildx build \
  --platform ${PLATFORMS} \
  --tag ${BUNDLE_IMG} \
  --file bundle.Dockerfile \
  --push \
  --progress=plain \
  .

echo "✅ Bundle image built and pushed"
docker buildx imagetools inspect ${BUNDLE_IMG}

echo ""
echo "=========================================="
echo "Step 4: Building Catalog Image"
echo "=========================================="

# Create catalog directory
mkdir -p catalog

# Initialize catalog
opm init self-node-remediation \
  --default-channel=stable \
  --description=./README.md \
  --icon=./config/assets/snr_icon_blue.png \
  --output yaml \
  > catalog/index.yaml

# Render bundle
opm render ${BUNDLE_IMG} --output yaml >> catalog/index.yaml

# Add channel
cat <<EOF >> catalog/index.yaml
---
schema: olm.channel
package: self-node-remediation
name: stable
entries:
  - name: self-node-remediation.v${VERSION}
EOF

# Validate catalog
opm validate catalog

# Generate Dockerfile
opm generate dockerfile catalog

# Build catalog image
docker buildx build \
  --platform ${PLATFORMS} \
  --tag ${CATALOG_IMG} \
  --file catalog.Dockerfile \
  --push \
  --progress=plain \
  .

echo "✅ Catalog image built and pushed"
docker buildx imagetools inspect ${CATALOG_IMG}

echo ""
echo "=========================================="
echo "✅ All Images Built Successfully!"
echo "=========================================="
echo ""
echo "Operator Image:"
echo "  ${IMG}"
docker buildx imagetools inspect ${IMG} | grep -E "Name:|Platform:"
echo ""
echo "Bundle Image:"
echo "  ${BUNDLE_IMG}"
docker buildx imagetools inspect ${BUNDLE_IMG} | grep -E "Name:|Platform:"
echo ""
echo "Catalog Image:"
echo "  ${CATALOG_IMG}"
docker buildx imagetools inspect ${CATALOG_IMG} | grep -E "Name:|Platform:"
echo ""
echo "=========================================="
echo "Next Steps:"
echo "1. Deploy using catalog: see DEPLOYMENT_GUIDE_S390X.md"
echo "2. Test on different architectures"
echo "3. Run E2E tests"
echo "=========================================="
```

### Save and Run the Script

```bash
# Save as build-multiarch.sh
cat > build-multiarch.sh << 'EOF'
[paste script above]
EOF

# Make executable
chmod +x build-multiarch.sh

# Run with default settings (amd64 + s390x)
./build-multiarch.sh

# Or customize platforms
export PLATFORMS=linux/amd64,linux/s390x,linux/arm64,linux/ppc64le
./build-multiarch.sh
```

---

## Troubleshooting

### Issue 1: QEMU Not Registered

**Symptom:**
```
ERROR: failed to solve: process "/bin/sh -c ..." did not complete successfully
```

**Solution:**
```bash
# Register QEMU handlers
docker run --rm --privileged multiarch/qemu-user-static --reset -p yes

# Verify
ls -la /proc/sys/fs/binfmt_misc/ | grep qemu
```

### Issue 2: Builder Not Supporting Platform

**Symptom:**
```
ERROR: multiple platforms feature is currently not supported for docker driver
```

**Solution:**
```bash
# Create a new builder with docker-container driver
docker buildx create --name multiarch-builder --driver docker-container --use

# Bootstrap it
docker buildx inspect --bootstrap

# Verify platforms
docker buildx inspect multiarch-builder | grep Platforms
```

### Issue 3: Cannot Load Multi-Platform Image

**Symptom:**
```
ERROR: docker exporter does not currently support exporting manifest lists
```

**Solution:**
```bash
# Use --push instead of --load for multi-platform builds
docker buildx build --platform linux/amd64,linux/s390x --push ...

# For local testing, build single platform with --load
docker buildx build --platform linux/amd64 --load ...
```

### Issue 4: Slow Build Times

**Symptom:**
Builds take very long, especially for non-native architectures.

**Solution:**
```bash
# Use build cache
docker buildx build \
  --platform linux/amd64,linux/s390x \
  --cache-from type=registry,ref=${IMG}-cache \
  --cache-to type=registry,ref=${IMG}-cache,mode=max \
  --push \
  .

# Build native architecture first, then others
docker buildx build --platform linux/$(uname -m) --push .
docker buildx build --platform linux/amd64,linux/s390x --push .
```

### Issue 5: Authentication Errors

**Symptom:**
```
ERROR: failed to push: unexpected status: 401 Unauthorized
```

**Solution:**
```bash
# Login to registry
docker login quay.io

# Verify credentials
cat ~/.docker/config.json | jq '.auths'

# For buildx, ensure credentials are available
docker buildx build --push --secret id=dockerconfig,src=$HOME/.docker/config.json .
```

### Issue 6: Exec Format Error in Built Image

**Symptom:**
```
exec /manager: exec format error
```

**Solution:**
```bash
# Verify TARGETARCH is being used in Dockerfile
grep TARGETARCH Dockerfile

# Verify hack/build.sh uses TARGETARCH
grep TARGETARCH hack/build.sh

# Check built image architecture
docker buildx imagetools inspect ${IMG} | grep Platform

# Pull specific architecture and test
docker pull --platform linux/s390x ${IMG}
docker run --rm --platform linux/s390x ${IMG} /manager --version
```

### Issue 7: Bundle Validation Fails

**Symptom:**
```
ERROR: bundle validation failed
```

**Solution:**
```bash
# Check bundle structure
ls -la bundle/manifests/
ls -la bundle/metadata/

# Validate manually
operator-sdk bundle validate ./bundle --select-optional suite=operatorframework

# Check CSV syntax
cat bundle/manifests/self-node-remediation.clusterserviceversion.yaml | yq eval

# Ensure correct operator image in CSV
grep "image:" bundle/manifests/self-node-remediation.clusterserviceversion.yaml
```

### Issue 8: Catalog Build Fails

**Symptom:**
```
ERROR: failed to render bundle
```

**Solution:**
```bash
# Ensure bundle image is pushed and accessible
docker pull ${BUNDLE_IMG}

# Validate bundle image
docker run --rm ${BUNDLE_IMG} ls -la /manifests/

# Check opm version
opm version

# Rebuild catalog from scratch
rm -rf catalog
mkdir catalog
opm init self-node-remediation --default-channel=stable --output yaml > catalog/index.yaml
opm render ${BUNDLE_IMG} --output yaml >> catalog/index.yaml
```

---

## Advanced Topics

### Building with GitHub Actions

Create `.github/workflows/multiarch-build.yml`:

```yaml
name: Multi-Arch Build

on:
  push:
    branches: [ main ]
    tags: [ 'v*' ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up QEMU
        uses: docker/setup-qemu-action@v2
        
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2
        
      - name: Login to Quay.io
        uses: docker/login-action@v2
        with:
          registry: quay.io
          username: ${{ secrets.QUAY_USERNAME }}
          password: ${{ secrets.QUAY_PASSWORD }}
          
      - name: Build and push
        uses: docker/build-push-action@v4
        with:
          context: .
          platforms: linux/amd64,linux/s390x,linux/arm64,linux/ppc64le
          push: true
          tags: quay.io/${{ secrets.QUAY_USERNAME }}/self-node-remediation-operator:${{ github.ref_name }}
```

### Using Makefile Targets

Add to `Makefile`:

```makefile
.PHONY: docker-buildx
docker-buildx: ## Build multi-arch image using buildx
	docker buildx build \
		--platform linux/amd64,linux/s390x \
		--tag $(IMG) \
		--file Dockerfile \
		--push \
		.

.PHONY: docker-buildx-all
docker-buildx-all: ## Build for all supported architectures
	docker buildx build \
		--platform linux/amd64,linux/s390x,linux/arm64,linux/ppc64le \
		--tag $(IMG) \
		--file Dockerfile \
		--push \
		.
```

Usage:
```bash
make docker-buildx IMG=quay.io/user/operator:v1.0
```

---

## Summary

### Key Takeaways

1. **Docker Buildx** enables building for multiple architectures from a single machine
2. **QEMU emulation** allows cross-compilation without native hardware
3. **Manifest lists** provide a single image reference for all architectures
4. **`--push` flag** is required for multi-platform builds (can't use `--load`)
5. **Architecture detection** in Dockerfile and build scripts is crucial

### Quick Reference

```bash
# Setup
docker buildx create --name multiarch --use
docker buildx inspect --bootstrap

# Build operator for multiple architectures
docker buildx build \
  --platform linux/amd64,linux/s390x \
  --tag ${IMG} \
  --push \
  .

# Inspect multi-arch image
docker buildx imagetools inspect ${IMG}

# Pull specific architecture
docker pull --platform linux/s390x ${IMG}
```

### Best Practices

1. ✅ Always use `--push` for multi-platform builds
2. ✅ Test each architecture after building
3. ✅ Use manifest lists for unified image references
4. ✅ Implement proper architecture detection in build scripts
5. ✅ Cache builds to improve performance
6. ✅ Validate images on target architectures when possible

---

**Document Version**: 1.0  
**Last Updated**: 2026-05-08  
**Purpose**: Comprehensive guide for cross-platform multi-architecture builds using Docker buildx
