# Deploying Self Node Remediation Operator on s390x OCP Cluster

This guide provides step-by-step instructions for deploying your custom s390x Self Node Remediation (SNR) Operator image on an OpenShift Container Platform (OCP) cluster running on s390x architecture, and integrating it with Node Health Check (NHC) Operator for E2E testing.

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Push Your Custom Image](#push-your-custom-image)
3. [Deployment Methods](#deployment-methods)
4. [Method 1: Deploy Using Kustomize (Recommended)](#method-1-deploy-using-kustomize-recommended)
5. [Method 2: Deploy Using OLM Bundle](#method-2-deploy-using-olm-bundle)
6. [Method 3: Deploy Using Manifests Directly](#method-3-deploy-using-manifests-directly)
7. [Verify Deployment](#verify-deployment)
8. [Install Node Health Check Operator](#install-node-health-check-operator)
9. [Configure NHC with SNR](#configure-nhc-with-snr)
10. [E2E Testing](#e2e-testing)
11. [Troubleshooting](#troubleshooting)

---

## Prerequisites

### 1. Access to OCP Cluster
```bash
# Login to your OCP cluster
oc login https://api.your-cluster.example.com:6443 --token=YOUR_TOKEN

# Verify you're on s390x
oc get nodes -o wide
# Check the ARCHITECTURE column shows s390x
```

### 2. Cluster Admin Permissions
```bash
# Verify you have cluster-admin role
oc auth can-i '*' '*' --all-namespaces
# Should return: yes
```

### 3. Custom Image Ready
```bash
# Your custom s390x image should be built and available
export IMG=quay.io/apriyada/self-node-remediation-operator:s390x
```

---

## Push Your Custom Image

### Step 1: Login to Quay.io
```bash
docker login quay.io
# Or with podman:
podman login quay.io
```

### Step 2: Push the Image
```bash
# Push your s390x image
docker push quay.io/apriyada/self-node-remediation-operator:s390x

# Verify the push
docker manifest inspect quay.io/apriyada/self-node-remediation-operator:s390x
```

### Step 3: Make Repository Public (if needed)
If your repository is private, either:
- Make it public on Quay.io, OR
- Create an image pull secret (see below)

#### Create Image Pull Secret (for private repos):
```bash
# Create secret in the operator namespace
oc create secret docker-registry quay-pull-secret \
  --docker-server=quay.io \
  --docker-username=apriyada \
  --docker-password=YOUR_PASSWORD \
  -n self-node-remediation

# Link the secret to the service account
oc secrets link self-node-remediation-controller-manager quay-pull-secret \
  --for=pull \
  -n self-node-remediation
```

---

## Deployment Methods

You have three options to deploy SNR on your OCP cluster:

1. **Kustomize** (Recommended) - Uses the config/ directory
2. **OLM Bundle** - Uses Operator Lifecycle Manager
3. **Direct Manifests** - Manual deployment

---

## Method 1: Deploy Using Kustomize (Recommended)

This is the easiest method for development and testing.

### Step 1: Set Your Image
```bash
# Set your custom image
export IMG=quay.io/apriyada/self-node-remediation-operator:s390x

# Update the image in kustomization
cd config/manager
kustomize edit set image controller=${IMG}
cd ../..
```

### Step 2: Create Namespace
```bash
# Create the namespace
oc create namespace self-node-remediation

# Label the namespace for privileged workloads (required for SNR)
oc label namespace self-node-remediation \
  security.openshift.io/scc.podSecurityLabelSync=false \
  pod-security.kubernetes.io/enforce=privileged \
  pod-security.kubernetes.io/audit=privileged \
  pod-security.kubernetes.io/warn=privileged
```

### Step 3: Deploy Using Make
```bash
# Deploy everything (CRDs, RBAC, Operator)
make deploy IMG=${IMG}

# This will:
# - Install CRDs
# - Create RBAC resources
# - Deploy the operator
# - Set up webhooks
```

### Step 4: Verify Deployment
```bash
# Check if operator is running
oc get pods -n self-node-remediation

# Expected output:
# NAME                                                        READY   STATUS    RESTARTS   AGE
# self-node-remediation-controller-manager-xxxxxxxxxx-xxxxx   2/2     Running   0          1m
```

---

## Method 2: Deploy Using OLM Bundle

This method uses the Operator Lifecycle Manager (OLM) which is the standard way to deploy operators on OpenShift.

### Step 1: Build and Push Bundle Image
```bash
# Set your image registry
export IMAGE_REGISTRY=quay.io/apriyada
export VERSION=0.1.0
export IMG=${IMAGE_REGISTRY}/self-node-remediation-operator:s390x
export BUNDLE_IMG=${IMAGE_REGISTRY}/self-node-remediation-operator-bundle:s390x

# Generate bundle manifests
make bundle VERSION=${VERSION}

# Build bundle image for s390x
docker build -f bundle.Dockerfile -t ${BUNDLE_IMG} .

# Push bundle image
docker push ${BUNDLE_IMG}
```

### Step 2: Deploy Using Operator SDK
```bash
# Create namespace
oc create namespace self-node-remediation

# Label namespace for privileged workloads
oc label namespace self-node-remediation \
  security.openshift.io/scc.podSecurityLabelSync=false \
  pod-security.kubernetes.io/enforce=privileged

# Deploy using operator-sdk
operator-sdk run bundle ${BUNDLE_IMG} -n self-node-remediation

# Or use the Makefile target
make bundle-run BUNDLE_IMG=${BUNDLE_IMG}
```

### Step 3: Verify OLM Deployment
```bash
# Check CSV (ClusterServiceVersion)
oc get csv -n self-node-remediation

# Check operator pod
oc get pods -n self-node-remediation

# Check subscription
oc get subscription -n self-node-remediation
```

---

## Method 3: Deploy Using Manifests Directly

This method gives you full control over the deployment.

### Step 1: Generate Manifests
```bash
# Generate all manifests
export IMG=quay.io/apriyada/self-node-remediation-operator:s390x
kustomize build config/default > snr-deployment.yaml
```

### Step 2: Update Image in Manifest
```bash
# Replace the image in the generated manifest
sed -i "s|controller:latest|${IMG}|g" snr-deployment.yaml
```

### Step 3: Create Namespace and Label It
```bash
# Create namespace
oc create namespace self-node-remediation

# Label for privileged workloads
oc label namespace self-node-remediation \
  security.openshift.io/scc.podSecurityLabelSync=false \
  pod-security.kubernetes.io/enforce=privileged
```

### Step 4: Apply Manifests
```bash
# Apply all resources
oc apply -f snr-deployment.yaml

# Or apply step by step:
# 1. CRDs
oc apply -f config/crd/bases/

# 2. RBAC
oc apply -f config/rbac/

# 3. Manager
oc apply -f config/manager/

# 4. Webhook
oc apply -f config/webhook/
```

---

## Verify Deployment

### Check Operator Status
```bash
# Check pods
oc get pods -n self-node-remediation

# Check logs
oc logs -n self-node-remediation \
  deployment/self-node-remediation-controller-manager \
  -c manager -f

# Check CRDs
oc get crd | grep self-node-remediation

# Expected CRDs:
# selfnoderemediationconfigs.self-node-remediation.medik8s.io
# selfnoderemediations.self-node-remediation.medik8s.io
# selfnoderemediationtemplates.self-node-remediation.medik8s.io
```

### Verify Image Architecture
```bash
# Check the running pod's image
oc get pod -n self-node-remediation \
  -l control-plane=controller-manager \
  -o jsonpath='{.items[0].spec.containers[0].image}'

# Should show: quay.io/apriyada/self-node-remediation-operator:s390x

# Verify it's running (no exec format errors)
oc get pods -n self-node-remediation
# STATUS should be "Running", not "CrashLoopBackOff"
```

### Create a Test SNR Config
```bash
# Create a basic SNR config
cat <<EOF | oc apply -f -
apiVersion: self-node-remediation.medik8s.io/v1alpha1
kind: SelfNodeRemediationConfig
metadata:
  name: self-node-remediation-config
  namespace: self-node-remediation
spec:
  watchdogFilePath: /dev/watchdog
  isSoftwareRebootEnabled: true
  apiCheckInterval: 15s
  apiServerTimeout: 5s
  maxApiErrorThreshold: 3
  peerApiServerTimeout: 5s
  peerDialTimeout: 5s
  peerRequestTimeout: 5s
  peerUpdateInterval: 15m
EOF

# Verify it was created
oc get selfnoderemediationconfig -n self-node-remediation
```

---

## Install Node Health Check Operator

NHC (Node Health Check) Operator monitors node health and triggers remediation using SNR.

### Method 1: Install via OperatorHub (Recommended for OCP)

```bash
# Create namespace for NHC
oc create namespace openshift-workload-availability

# Create OperatorGroup
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: openshift-workload-availability
  namespace: openshift-workload-availability
spec:
  targetNamespaces:
  - openshift-workload-availability
EOF

# Create Subscription
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: node-healthcheck-operator
  namespace: openshift-workload-availability
spec:
  channel: stable
  name: node-healthcheck-operator
  source: community-operators
  sourceNamespace: openshift-marketplace
EOF

# Wait for operator to be ready
oc get csv -n openshift-workload-availability -w
```

### Method 2: Install via Manifests

```bash
# Clone NHC operator repository
git clone https://github.com/medik8s/node-healthcheck-operator.git
cd node-healthcheck-operator

# Deploy NHC
make deploy IMG=quay.io/medik8s/node-healthcheck-operator:latest

# Verify deployment
oc get pods -n openshift-workload-availability
```

---

## Configure NHC with SNR

### Step 1: Create SNR Template

```bash
# Create a SelfNodeRemediationTemplate
cat <<EOF | oc apply -f -
apiVersion: self-node-remediation.medik8s.io/v1alpha1
kind: SelfNodeRemediationTemplate
metadata:
  name: self-node-remediation-template
  namespace: self-node-remediation
spec:
  template:
    spec:
      remediationStrategy: ResourceDeletion
EOF

# Verify template
oc get selfnoderemediationtemplate -n self-node-remediation
```

### Step 2: Create NodeHealthCheck Resource

```bash
# Create NHC that uses SNR for remediation
cat <<EOF | oc apply -f -
apiVersion: remediation.medik8s.io/v1alpha1
kind: NodeHealthCheck
metadata:
  name: nhc-worker-default
spec:
  minHealthy: 51%
  remediationTemplate:
    apiVersion: self-node-remediation.medik8s.io/v1alpha1
    kind: SelfNodeRemediationTemplate
    name: self-node-remediation-template
    namespace: self-node-remediation
  selector:
    matchExpressions:
      - key: node-role.kubernetes.io/worker
        operator: Exists
  unhealthyConditions:
    - type: Ready
      status: "False"
      duration: 300s
    - type: Ready
      status: Unknown
      duration: 300s
EOF

# Verify NHC
oc get nodehealthcheck
```

### Step 3: Verify Integration

```bash
# Check NHC status
oc get nodehealthcheck nhc-worker-default -o yaml

# Check if NHC can see SNR template
oc get selfnoderemediationtemplate -n self-node-remediation

# Check NHC operator logs
oc logs -n openshift-workload-availability \
  deployment/node-healthcheck-controller-manager \
  -c manager -f
```

---

## E2E Testing

### Test Scenario 1: Simulate Node Failure

```bash
# 1. Choose a worker node to test
export TEST_NODE=$(oc get nodes -l node-role.kubernetes.io/worker -o jsonpath='{.items[0].metadata.name}')
echo "Testing on node: ${TEST_NODE}"

# 2. Check current node status
oc get node ${TEST_NODE}

# 3. Simulate node failure by making it NotReady
# SSH to the node and stop kubelet
oc debug node/${TEST_NODE}
# In the debug pod:
chroot /host
systemctl stop kubelet
exit

# 4. Wait for NHC to detect the unhealthy node (5 minutes)
watch oc get nodes

# 5. Check if SNR remediation is triggered
oc get selfnoderemediation -A

# 6. Check NHC status
oc get nodehealthcheck nhc-worker-default -o yaml

# 7. Monitor SNR operator logs
oc logs -n self-node-remediation \
  deployment/self-node-remediation-controller-manager \
  -c manager -f

# 8. Verify remediation actions
oc get events -n self-node-remediation --sort-by='.lastTimestamp'
```

### Test Scenario 2: Workload Recovery

```bash
# 1. Deploy a test workload
cat <<EOF | oc apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-workload
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: test
  template:
    metadata:
      labels:
        app: test
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
EOF

# 2. Check pod distribution
oc get pods -n default -o wide -l app=test

# 3. Simulate node failure (as in Test Scenario 1)

# 4. Verify workloads are rescheduled to healthy nodes
watch oc get pods -n default -o wide -l app=test

# 5. Check remediation timeline
oc get events -A --sort-by='.lastTimestamp' | grep -i remediation
```

### Test Scenario 3: API Server Connectivity Test

```bash
# 1. Create SNR config with API check enabled
cat <<EOF | oc apply -f -
apiVersion: self-node-remediation.medik8s.io/v1alpha1
kind: SelfNodeRemediationConfig
metadata:
  name: self-node-remediation-config
  namespace: self-node-remediation
spec:
  watchdogFilePath: /dev/watchdog
  isSoftwareRebootEnabled: true
  apiCheckInterval: 15s
  apiServerTimeout: 5s
  maxApiErrorThreshold: 3
  peerApiServerTimeout: 5s
  peerDialTimeout: 5s
  peerRequestTimeout: 5s
  peerUpdateInterval: 15m
EOF

# 2. Monitor SNR agent logs on nodes
oc get pods -n self-node-remediation -o wide

# 3. Check API connectivity from SNR agents
oc logs -n self-node-remediation daemonset/self-node-remediation-agent -f
```

### Verify E2E Test Results

```bash
# 1. Check all SNR resources
oc get selfnoderemediation -A
oc get selfnoderemediationconfig -A
oc get selfnoderemediationtemplate -A

# 2. Check NHC status
oc get nodehealthcheck -o yaml

# 3. Check node status
oc get nodes

# 4. Review events
oc get events -A --sort-by='.lastTimestamp' | tail -50

# 5. Check operator logs
oc logs -n self-node-remediation \
  deployment/self-node-remediation-controller-manager \
  -c manager --tail=100
```

---

## Troubleshooting

### Issue 1: Operator Pod CrashLoopBackOff

```bash
# Check pod status
oc get pods -n self-node-remediation

# Check logs
oc logs -n self-node-remediation \
  deployment/self-node-remediation-controller-manager \
  -c manager

# Common causes:
# 1. Wrong architecture (exec format error)
#    Solution: Rebuild image with correct GOARCH=s390x

# 2. Missing RBAC permissions
#    Solution: Reapply RBAC manifests
oc apply -f config/rbac/

# 3. Webhook certificate issues
#    Solution: Check webhook configuration
oc get validatingwebhookconfiguration
oc get mutatingwebhookconfiguration
```

### Issue 2: Image Pull Errors

```bash
# Check image pull status
oc describe pod -n self-node-remediation \
  -l control-plane=controller-manager

# If ImagePullBackOff:
# 1. Verify image exists
docker manifest inspect quay.io/apriyada/self-node-remediation-operator:s390x

# 2. Check if repository is private
#    Create image pull secret (see Prerequisites section)

# 3. Verify secret is linked
oc get secrets -n self-node-remediation
oc describe sa self-node-remediation-controller-manager -n self-node-remediation
```

### Issue 3: NHC Not Triggering SNR

```bash
# 1. Check NHC status
oc get nodehealthcheck -o yaml

# 2. Verify SNR template exists
oc get selfnoderemediationtemplate -n self-node-remediation

# 3. Check NHC operator logs
oc logs -n openshift-workload-availability \
  deployment/node-healthcheck-controller-manager \
  -c manager -f

# 4. Verify node selector matches
oc get nodes --show-labels | grep worker

# 5. Check unhealthy conditions duration
#    Default is 300s (5 minutes) - wait longer if needed
```

### Issue 4: Remediation Not Working

```bash
# 1. Check SNR config
oc get selfnoderemediationconfig -n self-node-remediation -o yaml

# 2. Check SNR agent pods (DaemonSet)
oc get pods -n self-node-remediation -l app=self-node-remediation-agent

# 3. Check agent logs
oc logs -n self-node-remediation \
  daemonset/self-node-remediation-agent -f

# 4. Verify watchdog device exists on nodes
oc debug node/NODENAME
chroot /host
ls -l /dev/watchdog*

# 5. Check if software reboot is enabled
oc get selfnoderemediationconfig -n self-node-remediation \
  -o jsonpath='{.items[0].spec.isSoftwareRebootEnabled}'
```

### Issue 5: Webhook Failures

```bash
# Check webhook configuration
oc get validatingwebhookconfiguration | grep self-node-remediation
oc get mutatingwebhookconfiguration | grep self-node-remediation

# Check webhook service
oc get svc -n self-node-remediation

# Check webhook pod
oc get pods -n self-node-remediation -l control-plane=controller-manager

# Test webhook
oc create -f config/samples/self-node-remediation_v1alpha1_selfnoderemediation.yaml

# If webhook fails, check logs
oc logs -n self-node-remediation \
  deployment/self-node-remediation-controller-manager \
  -c manager | grep webhook
```

---

## Cleanup

### Remove SNR Operator

```bash
# Using make
make undeploy

# Or manually
oc delete namespace self-node-remediation

# Remove CRDs
oc delete crd selfnoderemediationconfigs.self-node-remediation.medik8s.io
oc delete crd selfnoderemediations.self-node-remediation.medik8s.io
oc delete crd selfnoderemediationtemplates.self-node-remediation.medik8s.io
```

### Remove NHC Operator

```bash
# If installed via OLM
oc delete subscription node-healthcheck-operator -n openshift-workload-availability
oc delete csv -n openshift-workload-availability -l operators.coreos.com/node-healthcheck-operator.openshift-workload-availability

# Remove namespace
oc delete namespace openshift-workload-availability
```

---

## Useful Commands Reference

```bash
# Check all SNR resources
oc get selfnoderemediation,selfnoderemediationconfig,selfnoderemediationtemplate -A

# Check operator status
oc get pods,svc,deploy -n self-node-remediation

# Watch node status
watch oc get nodes

# Monitor events
oc get events -A --sort-by='.lastTimestamp' -w

# Check logs
oc logs -n self-node-remediation deployment/self-node-remediation-controller-manager -c manager -f

# Describe resources
oc describe nodehealthcheck
oc describe selfnoderemediationconfig -n self-node-remediation

# Get resource YAML
oc get selfnoderemediation -A -o yaml
oc get nodehealthcheck -o yaml
```

---

## Additional Resources

- [Self Node Remediation Documentation](https://www.medik8s.io/remediation/self-node-remediation/self_node_remediation/)
- [Node Health Check Documentation](https://www.medik8s.io/design/node-healthcheck/)
- [Medik8s Project](https://www.medik8s.io/)
- [OpenShift Documentation](https://docs.openshift.com/)

---

## Summary

You now have:
1. ✅ Custom s390x SNR operator image deployed
2. ✅ Node Health Check operator installed
3. ✅ NHC configured to use SNR for remediation
4. ✅ E2E test scenarios to validate workload availability

Your s390x OCP cluster is now protected with automated node remediation!