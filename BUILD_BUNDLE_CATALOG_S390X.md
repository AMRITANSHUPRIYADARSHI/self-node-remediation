# Building Operator, Bundle, and Catalog Images for s390x

This guide provides complete step-by-step instructions for building all required images (operator, bundle, and catalog/index) for the Self Node Remediation operator on s390x architecture.

## Table of Contents
1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Step 1: Build Operator Image](#step-1-build-operator-image)
4. [Step 2: Build Bundle Image](#step-2-build-bundle-image)
5. [Step 3: Build Catalog/Index Image](#step-3-build-catalogindex-image)
6. [Step 4: Verify All Images](#step-4-verify-all-images)
7. [Step 5: Deploy Using Catalog](#step-5-deploy-using-catalog)
8. [Troubleshooting](#troubleshooting)

---

## Overview

To deploy an operator via OLM (Operator Lifecycle Manager) on OpenShift, you need three images:

1. **Operator Image** - The actual operator container
2. **Bundle Image** - Contains operator metadata (CSV, CRDs, etc.)
3. **Catalog/Index Image** - Registry of operator bundles

```
┌─────────────────────────────────────────────────────────┐
│                    Catalog Image                        │
│  (Index of available operator versions)                 │
│                                                          │
│  ┌────────────────────────────────────────────────┐    │
│  │              Bundle Image                      │    │
│  │  (Operator metadata: CSV, CRDs, RBAC)          │    │
│  │                                                 │    │
│  │  ┌──────────────────────────────────────────┐ │    │
│  │  │        Operator Image                    │ │    │
│  │  │  (The actual operator binary)            │ │    │
│  │  └──────────────────────────────────────────┘ │    │
│  └────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

---

## Prerequisites

### 1. Environment Setup

⚠️ **IMPORTANT:** You MUST set IMAGE_REGISTRY before running any commands, or you'll get "invalid reference format" errors!

```bash
# REQUIRED: Set your Quay.io registry (CHANGE THIS to your username!)
export IMAGE_REGISTRY=quay.io/apriyada

# Image tags (using s390x as the tag)
export IMG=${IMAGE_REGISTRY}/self-node-remediation-operator:s390x
export BUNDLE_IMG=${IMAGE_REGISTRY}/self-node-remediation-operator-bundle:s390x
export CATALOG_IMG=${IMAGE_REGISTRY}/self-node-remediation-operator-catalog:s390x

# VERSION is only for CSV metadata (not used in image tags)
# You can use any value - it only goes into the ClusterServiceVersion file
export VERSION=0.0.1

# Verify variables are set correctly
echo "IMAGE_REGISTRY: ${IMAGE_REGISTRY}"
echo "Operator Image: ${IMG}"
echo "Bundle Image: ${BUNDLE_IMG}"
echo "Catalog Image: ${CATALOG_IMG}"
echo "CSV Version (metadata only): ${VERSION}"

# If IMG shows "/self-node-remediation-operator:s390x" (missing registry),
# you forgot to export IMAGE_REGISTRY!
```

**Note:** Since you're using `s390x` as your image tag (not a version number), the VERSION variable is only used for the CSV metadata inside the bundle. Your image tags remain as `s390x`.

**Common Error:** If you see `Error: tag /self-node-remediation-operator:s390x: invalid reference format`, it means IMAGE_REGISTRY is not set. Always run `export IMAGE_REGISTRY=quay.io/YOUR_USERNAME` first!

### 2. Login to Quay.io

```bash
# Login with docker/podman
docker login quay.io
# Or
podman login quay.io

# Enter your credentials:
# Username: apriyada
# Password: <your-password-or-token>
```

### 3. Verify You're on s390x

```bash
# Check architecture
uname -m
# Should output: s390x

# Verify docker/podman is working
docker version
# Or
podman version
```

---

## Step 1: Build Operator Image

The operator image contains the actual operator binary.

### 1.1: Build the Operator Image

```bash
# IMPORTANT: Make sure IMAGE_REGISTRY is set!
echo "Building: ${IMG}"
# Should show: quay.io/apriyada/self-node-remediation-operator:s390x
# If it shows: /self-node-remediation-operator:s390x (missing registry),
# go back and export IMAGE_REGISTRY!

# Build operator image for s390x
make docker-build IMG=${IMG}

# This will:
# - Run tests
# - Build the Go binary for s390x
# - Create the container image
# - Tag it as: quay.io/apriyada/self-node-remediation-operator:s390x
```

### 1.2: Verify the Build

```bash
# Check the image was created
docker images | grep self-node-remediation-operator

# Verify architecture
docker inspect ${IMG} | grep Architecture
# Should show: "Architecture": "s390x"

# Check the binary inside
docker run --rm ${IMG} /manager --version
# Should run without "exec format error"
```

### 1.3: Push Operator Image

```bash
# Push to Quay.io
make docker-push IMG=${IMG}

# Or directly:
docker push ${IMG}

# Verify on Quay.io
echo "Check: https://quay.io/repository/apriyada/self-node-remediation-operator?tab=tags"
```

**✅ Checkpoint:** Operator image is built and pushed!

---

## Step 2: Build Bundle Image

The bundle image contains operator metadata required by OLM.

### 2.1: Generate Bundle Manifests

```bash
# Generate bundle manifests
# VERSION is only for CSV metadata (doesn't affect image tags)
make bundle VERSION=0.0.1

# This creates/updates:
# - bundle/manifests/self-node-remediation.clusterserviceversion.yaml (will have version: 0.0.1)
# - bundle/manifests/*.crd.yaml
# - bundle/metadata/annotations.yaml

# Note: The CSV will contain "version: 0.0.1" but your images will still use "s390x" tags
```

### 2.2: Review Generated Bundle

```bash
# Check bundle directory
ls -la bundle/

# Expected structure:
# bundle/
# ├── manifests/
# │   ├── self-node-remediation.clusterserviceversion.yaml
# │   ├── self-node-remediation.medik8s.io_selfnoderemediationconfigs.yaml
# │   ├── self-node-remediation.medik8s.io_selfnoderemediations.yaml
# │   ├── self-node-remediation.medik8s.io_selfnoderemediationtemplates.yaml
# │   └── ... (RBAC files)
# ├── metadata/
# │   └── annotations.yaml
# └── tests/
#     └── scorecard/

# Verify CSV references correct operator image
grep "image:" bundle/manifests/self-node-remediation.clusterserviceversion.yaml
# Should show your operator image
```

### 2.3: Update CSV with Correct Image

⚠️ **IMPORTANT:** The CSV file often contains the wrong image reference. You MUST update it!

```bash
# Check current image in CSV
grep "containerImage:" bundle/manifests/self-node-remediation.clusterserviceversion.yaml

# Expected output might show:
# containerImage: quay.io/medik8s/self-node-remediation-operator:latest

# Update to your s390x image:
# For Linux:
sed -i "s|containerImage:.*|containerImage: ${IMG}|" \
  bundle/manifests/self-node-remediation.clusterserviceversion.yaml

# For macOS:
sed -i '' "s|containerImage:.*|containerImage: ${IMG}|" \
  bundle/manifests/self-node-remediation.clusterserviceversion.yaml

# Or manually edit the file and change:
# FROM: containerImage: quay.io/medik8s/self-node-remediation-operator:latest
# TO:   containerImage: quay.io/apriyada/self-node-remediation-operator:s390x

# Verify the change:
grep "containerImage:" bundle/manifests/self-node-remediation.clusterserviceversion.yaml
# Should now show: containerImage: quay.io/apriyada/self-node-remediation-operator:s390x
```

**Also update the operator image in the deployment spec:**

```bash
# Check operator image in CSV deployment
grep -A 5 "image:" bundle/manifests/self-node-remediation.clusterserviceversion.yaml | grep -v "imagePullPolicy"

# Update all image references to your s390x image:
# For Linux:
sed -i "s|image:.*controller.*|image: ${IMG}|" \
  bundle/manifests/self-node-remediation.clusterserviceversion.yaml

# For macOS:
sed -i '' "s|image:.*controller.*|image: ${IMG}|" \
  bundle/manifests/self-node-remediation.clusterserviceversion.yaml

# Or manually edit and replace all occurrences of:
# FROM: image: quay.io/medik8s/self-node-remediation-operator:latest
# TO:   image: quay.io/apriyada/self-node-remediation-operator:s390x
```

### 2.4: Validate Bundle

```bash
# Validate the bundle
make bundle-validate

# Or manually:
operator-sdk bundle validate ./bundle

# Should show: "All validation tests have completed successfully"
```

### 2.5: Build Bundle Image for s390x

```bash
# Build bundle image
docker build -f bundle.Dockerfile -t ${BUNDLE_IMG} .

# Or use buildx for explicit s390x:
docker buildx build \
  --platform linux/s390x \
  -f bundle.Dockerfile \
  -t ${BUNDLE_IMG} \
  --load \
  .
```

### 2.6: Verify Bundle Image

```bash
# Check bundle image
docker images | grep bundle

# Inspect bundle contents
docker run --rm ${BUNDLE_IMG} ls -la /manifests/

# Should show CSV and CRD files
```

### 2.7: Push Bundle Image

```bash
# Push bundle to Quay.io
docker push ${BUNDLE_IMG}

# Verify on Quay.io
echo "Check: https://quay.io/repository/apriyada/self-node-remediation-operator-bundle?tab=tags"
```

**✅ Checkpoint:** Bundle image is built and pushed!

---

## Step 3: Build Catalog/Index Image

The catalog image is a registry of operator bundles that OLM uses to discover and install operators.

### 3.1: Build Catalog Image

```bash
# Build catalog using make target
make catalog-build CATALOG_IMG=${CATALOG_IMG}

# This will:
# - Create a catalog directory
# - Generate catalog index
# - Build the catalog image
```

### 3.2: Manual Catalog Build (Alternative)

If the make target doesn't work, build manually:

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

# Generate Dockerfile
opm generate dockerfile catalog

# Build catalog image
docker build -f catalog.Dockerfile -t ${CATALOG_IMG} .
```

### 3.3: Build Catalog for s390x

```bash
# If using buildx for explicit s390x:
docker buildx build \
  --platform linux/s390x \
  -f catalog.Dockerfile \
  -t ${CATALOG_IMG} \
  --load \
  .
```

### 3.4: Verify Catalog Image

```bash
# Check catalog image
docker images | grep catalog

# Inspect catalog contents
docker run --rm ${CATALOG_IMG} ls -la /

# Test catalog with opm
opm validate ${CATALOG_IMG}
```

### 3.5: Push Catalog Image

```bash
# Push catalog to Quay.io
docker push ${CATALOG_IMG}

# Verify on Quay.io
echo "Check: https://quay.io/repository/apriyada/self-node-remediation-operator-catalog?tab=tags"
```

**✅ Checkpoint:** Catalog image is built and pushed!

---

## Step 4: Verify All Images

### 4.1: Check All Images Exist

```bash
# List all images
docker images | grep self-node-remediation

# Expected output:
# quay.io/apriyada/self-node-remediation-operator          s390x    ...
# quay.io/apriyada/self-node-remediation-operator-bundle   s390x    ...
# quay.io/apriyada/self-node-remediation-operator-catalog  s390x    ...
```

### 4.2: Verify on Quay.io

```bash
# Check operator image
curl -s https://quay.io/api/v1/repository/apriyada/self-node-remediation-operator/tag/ | jq '.tags[] | select(.name=="s390x")'

# Check bundle image
curl -s https://quay.io/api/v1/repository/apriyada/self-node-remediation-operator-bundle/tag/ | jq '.tags[] | select(.name=="s390x")'

# Check catalog image
curl -s https://quay.io/api/v1/repository/apriyada/self-node-remediation-operator-catalog/tag/ | jq '.tags[] | select(.name=="s390x")'
```

### 4.3: Verify Architecture

```bash
# Check operator image architecture
docker manifest inspect ${IMG} | jq '.manifests[].platform'

# Check bundle image architecture
docker manifest inspect ${BUNDLE_IMG} | jq '.manifests[].platform'

# Check catalog image architecture
docker manifest inspect ${CATALOG_IMG} | jq '.manifests[].platform'

# All should show:
# {
#   "architecture": "s390x",
#   "os": "linux"
# }
```

---

## Step 5: Deploy Using Catalog

Now that all images are built and pushed, deploy the operator using the catalog.

### 5.1: Create CatalogSource

```bash
# Create catalog source in OpenShift
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: self-node-remediation-catalog
  namespace: openshift-marketplace
spec:
  sourceType: grpc
  image: ${CATALOG_IMG}
  displayName: Self Node Remediation Operator
  publisher: Medik8s
  updateStrategy:
    registryPoll:
      interval: 10m
EOF

# Verify catalog source
oc get catalogsource -n openshift-marketplace

# Wait for catalog to be ready
oc get catalogsource self-node-remediation-catalog -n openshift-marketplace -w
# Wait until READY shows "READY"
```

### 5.2: Create Namespace

```bash
# Create operator namespace
oc create namespace self-node-remediation

# Label for privileged workloads
oc label namespace self-node-remediation \
  security.openshift.io/scc.podSecurityLabelSync=false \
  pod-security.kubernetes.io/enforce=privileged
```

### 5.3: Create OperatorGroup

```bash
# Create operator group
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: self-node-remediation-operator-group
  namespace: self-node-remediation
spec:
  targetNamespaces:
  - self-node-remediation
EOF

# Verify operator group
oc get operatorgroup -n self-node-remediation
```

### 5.4: Create Subscription

```bash
# Create subscription
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: self-node-remediation-operator
  namespace: self-node-remediation
spec:
  channel: stable
  name: self-node-remediation
  source: self-node-remediation-catalog
  sourceNamespace: openshift-marketplace
  installPlanApproval: Automatic
EOF

# Verify subscription
oc get subscription -n self-node-remediation
```

### 5.5: Verify Installation

```bash
# Check install plan
oc get installplan -n self-node-remediation

# Check CSV (ClusterServiceVersion)
oc get csv -n self-node-remediation

# Wait for CSV to be successful
oc get csv -n self-node-remediation -w
# Wait until PHASE shows "Succeeded"

# Check operator pod
oc get pods -n self-node-remediation

# Check operator logs
oc logs -n self-node-remediation \
  deployment/self-node-remediation-controller-manager \
  -c manager -f
```

---

## Complete Build Script

Here's a complete script to build all images:

```bash
#!/bin/bash
set -e

# Configuration
export IMAGE_REGISTRY=quay.io/apriyada
export VERSION=0.8.0
export PREVIOUS_VERSION=0.7.0

# Image names
export IMG=${IMAGE_REGISTRY}/self-node-remediation-operator:s390x
export BUNDLE_IMG=${IMAGE_REGISTRY}/self-node-remediation-operator-bundle:s390x
export CATALOG_IMG=${IMAGE_REGISTRY}/self-node-remediation-operator-catalog:s390x

echo "=========================================="
echo "Building Self Node Remediation Operator"
echo "=========================================="
echo "Architecture: s390x"
echo "Version: ${VERSION}"
echo "Operator Image: ${IMG}"
echo "Bundle Image: ${BUNDLE_IMG}"
echo "Catalog Image: ${CATALOG_IMG}"
echo "=========================================="

# Step 1: Build and push operator image
echo "Step 1: Building operator image..."
make docker-build IMG=${IMG}
echo "Pushing operator image..."
make docker-push IMG=${IMG}
echo "✅ Operator image complete"

# Step 2: Generate bundle
echo "Step 2: Generating bundle..."
make bundle VERSION=${VERSION} PREVIOUS_VERSION=${PREVIOUS_VERSION}
echo "✅ Bundle manifests generated"

# Step 3: Build and push bundle image
echo "Step 3: Building bundle image..."
docker build -f bundle.Dockerfile -t ${BUNDLE_IMG} .
echo "Pushing bundle image..."
docker push ${BUNDLE_IMG}
echo "✅ Bundle image complete"

# Step 4: Build and push catalog image
echo "Step 4: Building catalog image..."
make catalog-build CATALOG_IMG=${CATALOG_IMG}
echo "Pushing catalog image..."
docker push ${CATALOG_IMG}
echo "✅ Catalog image complete"

# Verify all images
echo "=========================================="
echo "Verifying all images..."
echo "=========================================="
docker images | grep self-node-remediation

echo "=========================================="
echo "✅ All images built and pushed successfully!"
echo "=========================================="
echo "Operator: ${IMG}"
echo "Bundle: ${BUNDLE_IMG}"
echo "Catalog: ${CATALOG_IMG}"
echo "=========================================="
echo ""
echo "Next steps:"
echo "1. Create CatalogSource in OpenShift"
echo "2. Create Subscription to install operator"
echo "See DEPLOYMENT_GUIDE_S390X.md for details"
```

Save this as `build-all-s390x.sh` and run:

```bash
chmod +x build-all-s390x.sh
./build-all-s390x.sh
```

---

## Troubleshooting

### Issue 1: Bundle Validation Fails

```bash
# Check bundle structure
ls -la bundle/manifests/
ls -la bundle/metadata/

# Validate manually
operator-sdk bundle validate ./bundle --select-optional suite=operatorframework

# Common issues:
# 1. Missing CSV
#    Solution: Regenerate bundle with make bundle

# 2. Invalid CSV format
#    Solution: Check CSV syntax
cat bundle/manifests/self-node-remediation.clusterserviceversion.yaml | yq eval

# 3. Wrong operator image reference
#    Solution: Update CSV
sed -i "s|image:.*controller.*|image: ${IMG}|" \
  bundle/manifests/self-node-remediation.clusterserviceversion.yaml
```

### Issue 2: Catalog Build Fails

```bash
# Check if opm is installed
opm version

# If not installed, download for s390x
ARCH=$(uname -m)
case $ARCH in
  s390x) OPM_ARCH=s390x ;;
  *) echo "Unsupported arch: $ARCH" && exit 1 ;;
esac

curl -sSLo opm https://github.com/operator-framework/operator-registry/releases/download/v1.61.0/linux-${OPM_ARCH}-opm
chmod +x opm
sudo mv opm /usr/local/bin/

# Verify
opm version
```

### Issue 3: CatalogSource Not Ready

```bash
# Check catalog source status
oc get catalogsource self-node-remediation-catalog -n openshift-marketplace -o yaml

# Check catalog pod
oc get pods -n openshift-marketplace | grep self-node-remediation

# Check catalog pod logs
oc logs -n openshift-marketplace \
  $(oc get pods -n openshift-marketplace -l olm.catalogSource=self-node-remediation-catalog -o name) \
  -f

# Common issues:
# 1. Image pull error
#    Solution: Make sure catalog image is public or create image pull secret

# 2. Invalid catalog format
#    Solution: Validate catalog
opm validate catalog/

# 3. Wrong architecture
#    Solution: Rebuild catalog for s390x
```

### Issue 4: Subscription Not Installing

```bash
# Check subscription status
oc get subscription self-node-remediation-operator -n self-node-remediation -o yaml

# Check install plan
oc get installplan -n self-node-remediation -o yaml

# Check catalog source
oc get packagemanifest | grep self-node-remediation

# If package not found:
# 1. Wait for catalog to sync (can take 5-10 minutes)
# 2. Check catalog pod logs
# 3. Verify catalog image is correct
```

### Issue 5: Operator Pod CrashLoopBackOff

```bash
# Check pod status
oc get pods -n self-node-remediation

# Check pod logs
oc logs -n self-node-remediation \
  $(oc get pods -n self-node-remediation -l control-plane=controller-manager -o name) \
  -c manager

# Common issues:
# 1. Exec format error
#    Solution: Rebuild operator image with correct GOARCH=s390x

# 2. Missing RBAC
#    Solution: Check CSV has correct RBAC definitions

# 3. Webhook issues
#    Solution: Check webhook configuration in CSV
```

---

## Summary Checklist

Use this checklist to track your progress:

- [ ] **Operator Image**
  - [ ] Built for s390x
  - [ ] Pushed to Quay.io
  - [ ] Verified architecture
  - [ ] Tested binary runs without exec format error

- [ ] **Bundle Image**
  - [ ] Generated bundle manifests
  - [ ] Validated bundle
  - [ ] Built bundle image for s390x
  - [ ] Pushed to Quay.io
  - [ ] Verified CSV references correct operator image

- [ ] **Catalog Image**
  - [ ] Built catalog image for s390x
  - [ ] Validated catalog
  - [ ] Pushed to Quay.io
  - [ ] Verified catalog structure

- [ ] **Deployment**
  - [ ] Created CatalogSource
  - [ ] Created OperatorGroup
  - [ ] Created Subscription
  - [ ] Verified CSV is Succeeded
  - [ ] Verified operator pod is Running

---

## Quick Reference

```bash
# Build all images
make docker-build IMG=${IMG}
make bundle VERSION=${VERSION}
docker build -f bundle.Dockerfile -t ${BUNDLE_IMG} .
make catalog-build CATALOG_IMG=${CATALOG_IMG}

# Push all images
docker push ${IMG}
docker push ${BUNDLE_IMG}
docker push ${CATALOG_IMG}

# Deploy via catalog
oc apply -f catalogsource.yaml
oc apply -f operatorgroup.yaml
oc apply -f subscription.yaml

# Verify
oc get csv -n self-node-remediation
oc get pods -n self-node-remediation
```

---

## Additional Resources

- [Operator SDK Bundle Documentation](https://sdk.operatorframework.io/docs/olm-integration/tutorial-bundle/)
- [OLM Documentation](https://olm.operatorframework.io/)
- [OPM Documentation](https://github.com/operator-framework/operator-registry)
- [OpenShift Operator Documentation](https://docs.openshift.com/container-platform/latest/operators/understanding/olm/olm-understanding-olm.html)