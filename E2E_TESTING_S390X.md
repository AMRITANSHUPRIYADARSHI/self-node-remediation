# End-to-End Testing Guide for Self Node Remediation on s390x

This guide provides step-by-step instructions for running E2E tests with the Self Node Remediation operator on s390x architecture, integrated with Node Health Check (NHC) operator.

## 📋 Prerequisites

### 1. OpenShift Cluster on s390x
- OCP cluster running on IBM Z (s390x) architecture
- Cluster admin access
- `oc` CLI configured with cluster access

### 2. Install Required Tools

#### Install Go (s390x)
```bash
sudo rm -rf /usr/local/go
curl -LO https://go.dev/dl/go1.23.1.linux-s390x.tar.gz
sudo tar -C /usr/local -xzf go1.23.1.linux-s390x.tar.gz
echo "export PATH=$PATH:/usr/local/go/bin" >> ~/.bashrc
echo "export GOPATH=$HOME/go" >> ~/.bashrc
echo "export PATH=$PATH:$GOPATH/bin" >> ~/.bashrc
source ~/.bashrc
go version
```

#### Install Ginkgo
```bash
go install github.com/onsi/ginkgo/v2/ginkgo@v2.23.4
ginkgo version
```

#### Install Node.js and Yarn (for console plugin tests)
```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/master/install.sh | bash
source ~/.bashrc
command -v nvm
nvm install node
nvm use node
nvm alias default node
node -v
npm -v
sudo npm install -g yarn
yarn --version
```

### 3. Verify Operators are Installed

```bash
# Check Node Health Check Operator
oc get csv -n openshift-workload-availability | grep node-healthcheck

# Check Self Node Remediation Operator
oc get csv -n openshift-workload-availability | grep self-node-remediation

# Verify operator pods
oc get pods -n openshift-workload-availability
```

Expected output:
```
NAME                                                       READY   STATUS
node-healthcheck-controller-manager-xxxxx                  2/2     Running
self-node-remediation-controller-manager-xxxxx             2/2     Running
self-node-remediation-ds-xxxxx                             1/1     Running (on each node)
```

## 🧪 Running E2E Tests

### Recommended: Test via Node Health Check Operator Repository

This tests the integration between NHC and SNR operators.

#### 1. Clone Node Health Check Operator Repository
```bash
cd ~
git clone https://github.com/medik8s/node-healthcheck-operator.git
cd node-healthcheck-operator
```

#### 2. Set Environment Variables
```bash
export KUBECONFIG=/root/.kube/config
export SNR_STRATEGY="Automatic"
export OPERATOR_NS="openshift-workload-availability"

# Verify environment
echo "KUBECONFIG: $KUBECONFIG"
echo "SNR_STRATEGY: $SNR_STRATEGY"
echo "OPERATOR_NS: $OPERATOR_NS"

# Test cluster connection
oc whoami
oc get nodes
```

#### 3. Run E2E Tests
```bash
# Run all E2E tests
make test-e2e

# Or run specific test suites
ginkgo -v ./e2e/nhc_e2e_test.go
ginkgo -v ./e2e/nhc_console_e2e_test.go
```

#### 4. Expected Test Results

| Test File | Test Case | Expected Result |
|-----------|-----------|-----------------|
| **e2e/nhc_e2e_test.go** | should report disabled NHC | ✅ Pass |
| **e2e/nhc_e2e_test.go** | Worker node with unhealthy condition, with escalating remediation config | ✅ Pass |
| **e2e/nhc_e2e_test.go** | Worker node with unhealthy condition, with classic remediation config | ✅ Pass |
| **e2e/nhc_e2e_test.go** | Worker node with unhealthy condition, with terminating node | ✅ Pass |
| **e2e/nhc_e2e_test.go** | Control plane node with unhealthy condition, with classic remediation | ✅ Pass |
| **e2e/nhc_console_e2e_test.go** | Console plugin manifest should be served | ✅ Pass |
| **e2e/mhc_e2e_test.go** | MHC test | ⏭ Skipped |

### Alternative: Test via Self Node Remediation Repository

#### 1. Clone Self Node Remediation Repository
```bash
cd ~
git clone https://github.com/medik8s/self-node-remediation.git
cd self-node-remediation
```

#### 2. Set Environment Variables
```bash
export KUBECONFIG=/root/.kube/config
export OPERATOR_NS="openshift-workload-availability"

# Verify
oc whoami
oc get nodes
```

#### 3. Run E2E Tests
```bash
# Run all E2E tests
make test-e2e

# Or run with ginkgo directly
ginkgo -v ./e2e/
```

## 🐛 Troubleshooting

### Issue 1: Connection Refused (localhost:8080)

**Error:**
```
Get "http://localhost:8080/api?timeout=32s": dial tcp [::1]:8080: connect: connection refused
```

**Solution:**
```bash
# Ensure KUBECONFIG is exported
export KUBECONFIG=/root/.kube/config

# Verify it's working
oc whoami
oc get nodes

# Run tests again
make test-e2e
```

### Issue 2: Operator Not Ready

**Check operator status:**
```bash
oc get csv -n openshift-workload-availability
oc get pods -n openshift-workload-availability

# Check operator logs
oc logs -n openshift-workload-availability deployment/self-node-remediation-controller-manager -c manager --tail=100
```

### Issue 3: Test Timeout

```bash
# Increase timeout
export GINKGO_TIMEOUT=30m

# Run with increased timeout
ginkgo -v -timeout=30m ./e2e/
```

### Issue 4: Architecture Mismatch

```bash
# Verify operator is running s390x binary
oc logs -n openshift-workload-availability deployment/self-node-remediation-controller-manager -c manager | grep "Go OS/Arch"
```

Expected: `Go OS/Arch: linux/s390x`

## 🎯 Manual Testing Scenarios

### Scenario 1: Simulate Node Failure

```bash
# 1. Create a test NodeHealthCheck
cat <<EOF | oc apply -f -
apiVersion: remediation.medik8s.io/v1alpha1
kind: NodeHealthCheck
metadata:
  name: test-nhc
spec:
  minHealthy: 1
  remediationTemplate:
    apiVersion: self-node-remediation.medik8s.io/v1alpha1
    kind: SelfNodeRemediationTemplate
    name: self-node-remediation-automatic-strategy-template
    namespace: openshift-workload-availability
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

# 2. Watch for NHC status
oc get nodehealthcheck test-nhc -w

# 3. Simulate node failure (on a worker node)
# SSH to a worker node and stop kubelet
ssh core@<worker-node-ip>
sudo systemctl stop kubelet

# 4. Watch remediation
oc get selfnoderemediation -A -w
oc get nodes -w

# 5. Cleanup
oc delete nodehealthcheck test-nhc
```

### Scenario 2: Test DaemonSet Functionality

```bash
# Check DaemonSet is running on all nodes
oc get ds -n openshift-workload-availability self-node-remediation-ds

# Check DaemonSet pods
oc get pods -n openshift-workload-availability -l app=self-node-remediation-ds -o wide

# Check DaemonSet logs
oc logs -n openshift-workload-availability -l app=self-node-remediation-ds --tail=50
```

## 📊 Test Scenarios Explained

### 1. Disabled NHC Test
- **Purpose**: Verify NHC reports correctly when disabled
- **What it tests**: Status reporting and configuration handling

### 2. Worker Node with Unhealthy Condition (Escalating)
- **Purpose**: Test escalating remediation strategy
- **What it tests**: 
  - Node becomes unhealthy
  - SNR attempts remediation
  - If remediation fails, escalates to next strategy
  - Node is recovered or replaced

### 3. Worker Node with Unhealthy Condition (Classic)
- **Purpose**: Test classic remediation strategy
- **What it tests**:
  - Node becomes unhealthy
  - SNR performs immediate remediation
  - Node is recovered

### 4. Worker Node with Terminating Node
- **Purpose**: Test handling of terminating nodes
- **What it tests**:
  - Node enters terminating state
  - SNR handles graceful shutdown
  - Workloads are migrated

### 5. Control Plane Node with Unhealthy Condition
- **Purpose**: Test remediation of control plane nodes
- **What it tests**:
  - Control plane node becomes unhealthy
  - SNR performs safe remediation
  - Cluster stability maintained

### 6. Console Plugin Test
- **Purpose**: Verify console plugin integration
- **What it tests**:
  - Plugin manifest is served
  - Console integration works
  - UI components load correctly

## 📝 Test Report Template

```markdown
# Self Node Remediation E2E Test Report - s390x

**Date**: YYYY-MM-DD
**Cluster**: <cluster-name>
**Architecture**: s390x
**OCP Version**: <version>
**SNR Version**: <version>
**NHC Version**: <version>

## Test Results

| Test Suite | Test Case | Status | Duration | Notes |
|------------|-----------|--------|----------|-------|
| nhc_e2e_test.go | Disabled NHC | ✅/❌ | Xs | |
| nhc_e2e_test.go | Escalating remediation | ✅/❌ | Xs | |
| nhc_e2e_test.go | Classic remediation | ✅/❌ | Xs | |
| nhc_e2e_test.go | Terminating node | ✅/❌ | Xs | |
| nhc_e2e_test.go | Control plane remediation | ✅/❌ | Xs | |
| nhc_console_e2e_test.go | Console plugin | ✅/❌ | Xs | |

## Issues Found

1. [Issue description]
   - **Severity**: High/Medium/Low
   - **Workaround**: [if any]

## Recommendations

1. [Recommendation 1]
2. [Recommendation 2]
```

## 🔗 Additional Resources

- [Medik8s Documentation](https://www.medik8s.io/)
- [Node Health Check Operator](https://github.com/medik8s/node-healthcheck-operator)
- [Self Node Remediation Operator](https://github.com/medik8s/self-node-remediation)
- [Workload Availability for Red Hat OpenShift](https://docs.redhat.com/en/documentation/workload_availability_for_red_hat_openshift/)