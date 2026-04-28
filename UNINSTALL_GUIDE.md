# Complete Uninstallation Guide for Self Node Remediation & Node Health Check Operators

This guide provides step-by-step instructions to completely remove both the Self Node Remediation (SNR) operator and the Node Health Check (NHC) operator from your OpenShift cluster.

## Table of Contents

1. [Quick Uninstall Script](#quick-uninstall-script)
2. [Step-by-Step Manual Uninstall](#step-by-step-manual-uninstall)
3. [Verification](#verification)
4. [Cleanup Local Images](#cleanup-local-images)
5. [Troubleshooting](#troubleshooting)

---

## Quick Uninstall Script

### Complete Uninstall (Both SNR and NHC Operators)

```bash
#!/bin/bash
set -e

echo "=== Uninstalling Self Node Remediation & Node Health Check Operators ==="

# Set namespace
NAMESPACE="openshift-workload-availability"

echo ""
echo "=========================================="
echo "PART 1: Uninstalling Self Node Remediation Operator"
echo "=========================================="
echo ""

echo "Step 1: Deleting SNR Custom Resources..."
oc delete selfnoderemediation --all -n ${NAMESPACE} --ignore-not-found=true --timeout=60s
oc delete selfnoderemediationconfig --all -n ${NAMESPACE} --ignore-not-found=true --timeout=60s
oc delete selfnoderemediationtemplate --all -n ${NAMESPACE} --ignore-not-found=true --timeout=60s

echo "Step 2: Deleting SNR Subscription..."
oc delete subscription self-node-remediation-operator -n ${NAMESPACE} --ignore-not-found=true

echo "Step 3: Deleting SNR CSV..."
oc delete csv -l operators.coreos.com/self-node-remediation-operator.${NAMESPACE} -n ${NAMESPACE} --ignore-not-found=true
oc delete csv self-node-remediation.v0.0.1 -n ${NAMESPACE} --ignore-not-found=true

echo "Step 4: Deleting SNR InstallPlan..."
oc delete installplan -l operators.coreos.com/self-node-remediation-operator.${NAMESPACE} -n ${NAMESPACE} --ignore-not-found=true

echo "Step 5: Deleting SNR CatalogSource..."
oc delete catalogsource self-node-remediation-cs -n openshift-marketplace --ignore-not-found=true

echo "Step 6: Deleting SNR Deployments and DaemonSets..."
oc delete deployment self-node-remediation-controller-manager -n ${NAMESPACE} --ignore-not-found=true
oc delete daemonset self-node-remediation-ds -n ${NAMESPACE} --ignore-not-found=true

echo "Step 7: Deleting SNR Services..."
oc delete service self-node-remediation-controller-manager-metrics-service -n ${NAMESPACE} --ignore-not-found=true
oc delete service self-node-remediation-webhook-service -n ${NAMESPACE} --ignore-not-found=true

echo "Step 8: Deleting SNR ConfigMaps and Secrets..."
oc delete configmap -l app.kubernetes.io/name=self-node-remediation -n ${NAMESPACE} --ignore-not-found=true
oc delete secret -l app.kubernetes.io/name=self-node-remediation -n ${NAMESPACE} --ignore-not-found=true

echo "Step 9: Deleting SNR RBAC..."
oc delete clusterrolebinding self-node-remediation-controller-manager --ignore-not-found=true
oc delete clusterrolebinding self-node-remediation-manager-rolebinding --ignore-not-found=true
oc delete clusterrole self-node-remediation-controller-manager --ignore-not-found=true
oc delete clusterrole self-node-remediation-manager-role --ignore-not-found=true
oc delete serviceaccount self-node-remediation-controller-manager -n ${NAMESPACE} --ignore-not-found=true

echo ""
echo "=========================================="
echo "PART 2: Uninstalling Node Health Check Operator"
echo "=========================================="
echo ""

echo "Step 1: Deleting NHC Custom Resources..."
oc delete nodehealthcheck --all --ignore-not-found=true --timeout=60s

echo "Step 2: Deleting NHC Subscription..."
oc delete subscription node-healthcheck-operator -n ${NAMESPACE} --ignore-not-found=true

echo "Step 3: Deleting NHC CSV..."
oc delete csv -l operators.coreos.com/node-healthcheck-operator.${NAMESPACE} -n ${NAMESPACE} --ignore-not-found=true
# Try common NHC CSV names
oc delete csv node-healthcheck-operator.v0.8.0 -n ${NAMESPACE} --ignore-not-found=true
oc delete csv node-healthcheck-operator.v0.7.0 -n ${NAMESPACE} --ignore-not-found=true

echo "Step 4: Deleting NHC InstallPlan..."
oc delete installplan -l operators.coreos.com/node-healthcheck-operator.${NAMESPACE} -n ${NAMESPACE} --ignore-not-found=true

echo "Step 5: Deleting NHC CatalogSource (if custom)..."
oc delete catalogsource node-healthcheck-cs -n openshift-marketplace --ignore-not-found=true

echo "Step 6: Deleting NHC Deployments..."
oc delete deployment node-healthcheck-controller-manager -n ${NAMESPACE} --ignore-not-found=true

echo "Step 7: Deleting NHC Services..."
oc delete service node-healthcheck-controller-manager-metrics-service -n ${NAMESPACE} --ignore-not-found=true
oc delete service node-healthcheck-webhook-service -n ${NAMESPACE} --ignore-not-found=true

echo "Step 8: Deleting NHC ConfigMaps and Secrets..."
oc delete configmap -l app.kubernetes.io/name=node-healthcheck -n ${NAMESPACE} --ignore-not-found=true
oc delete secret -l app.kubernetes.io/name=node-healthcheck -n ${NAMESPACE} --ignore-not-found=true

echo "Step 9: Deleting NHC RBAC..."
oc delete clusterrolebinding node-healthcheck-controller-manager --ignore-not-found=true
oc delete clusterrolebinding node-healthcheck-manager-rolebinding --ignore-not-found=true
oc delete clusterrole node-healthcheck-controller-manager --ignore-not-found=true
oc delete clusterrole node-healthcheck-manager-role --ignore-not-found=true
oc delete serviceaccount node-healthcheck-controller-manager -n ${NAMESPACE} --ignore-not-found=true

echo ""
echo "=========================================="
echo "PART 3: Cleanup CRDs (Optional)"
echo "=========================================="
echo ""

read -p "Do you want to delete CRDs? This will affect all namespaces. (y/N): " -n 1 -r
echo
if [[ $REPLY =~ ^[Yy]$ ]]; then
    echo "Deleting SNR CRDs..."
    oc delete crd selfnoderemediationconfigs.self-node-remediation.medik8s.io --ignore-not-found=true
    oc delete crd selfnoderemediations.self-node-remediation.medik8s.io --ignore-not-found=true
    oc delete crd selfnoderemediationtemplates.self-node-remediation.medik8s.io --ignore-not-found=true
    
    echo "Deleting NHC CRDs..."
    oc delete crd nodehealthchecks.remediation.medik8s.io --ignore-not-found=true
    
    echo "CRDs deleted."
else
    echo "CRDs kept."
fi

echo ""
echo "=========================================="
echo "PART 4: Cleanup OperatorGroup (Optional)"
echo "=========================================="
echo ""

# Check if other operators exist in the namespace
OTHER_OPERATORS=$(oc get subscription -n ${NAMESPACE} --no-headers 2>/dev/null | wc -l)

if [ "$OTHER_OPERATORS" -eq 0 ]; then
    read -p "No other operators found. Delete OperatorGroup? (y/N): " -n 1 -r
    echo
    if [[ $REPLY =~ ^[Yy]$ ]]; then
        oc delete operatorgroup -n ${NAMESPACE} --all --ignore-not-found=true
        echo "OperatorGroup deleted."
    else
        echo "OperatorGroup kept."
    fi
else
    echo "Other operators found in namespace. Keeping OperatorGroup."
fi

echo ""
echo "=== Uninstallation Complete ==="
echo ""
echo "Verification:"
echo ""
echo "SNR Resources:"
oc get csv -n ${NAMESPACE} | grep self-node-remediation || echo "✓ No SNR CSV found"
oc get pods -n ${NAMESPACE} | grep self-node-remediation || echo "✓ No SNR pods found"
oc get crd | grep self-node-remediation || echo "✓ No SNR CRDs found"
echo ""
echo "NHC Resources:"
oc get csv -n ${NAMESPACE} | grep node-healthcheck || echo "✓ No NHC CSV found"
oc get pods -n ${NAMESPACE} | grep node-healthcheck || echo "✓ No NHC pods found"
oc get crd | grep nodehealthcheck || echo "✓ No NHC CRDs found"
```

### Save and Run

```bash
# Save the script
cat > uninstall-operators.sh << 'EOF'
[paste the script above]
EOF

# Make executable
chmod +x uninstall-operators.sh

# Run
./uninstall-operators.sh
```

---

## Step-by-Step Manual Uninstall

### PART 1: Self Node Remediation Operator

#### Step 1: Delete SNR Custom Resources

```bash
# List all SelfNodeRemediation resources
oc get selfnoderemediation -A

# Delete all in the namespace
oc delete selfnoderemediation --all -n openshift-workload-availability

# Delete SelfNodeRemediationConfig
oc get selfnoderemediationconfig -A
oc delete selfnoderemediationconfig --all -n openshift-workload-availability

# Delete SelfNodeRemediationTemplate
oc get selfnoderemediationtemplate -A
oc delete selfnoderemediationtemplate --all -n openshift-workload-availability
```

#### Step 2: Delete SNR Subscription

```bash
# List subscriptions
oc get subscription -n openshift-workload-availability

# Delete SNR subscription
oc delete subscription self-node-remediation-operator -n openshift-workload-availability

# Verify deletion
oc get subscription -n openshift-workload-availability | grep self-node-remediation
```

#### Step 3: Delete SNR CSV

```bash
# List CSVs
oc get csv -n openshift-workload-availability

# Delete SNR CSV
oc delete csv self-node-remediation.v0.0.1 -n openshift-workload-availability

# Or delete by label
oc delete csv -l operators.coreos.com/self-node-remediation-operator.openshift-workload-availability \
    -n openshift-workload-availability

# Verify deletion
oc get csv -n openshift-workload-availability | grep self-node-remediation
```

#### Step 4: Delete SNR InstallPlan

```bash
# List InstallPlans
oc get installplan -n openshift-workload-availability

# Delete SNR InstallPlans
oc delete installplan -l operators.coreos.com/self-node-remediation-operator.openshift-workload-availability \
    -n openshift-workload-availability

# Verify deletion
oc get installplan -n openshift-workload-availability
```

#### Step 5: Delete SNR CatalogSource

```bash
# List CatalogSources
oc get catalogsource -n openshift-marketplace

# Delete SNR CatalogSource
oc delete catalogsource self-node-remediation-cs -n openshift-marketplace

# Verify deletion
oc get catalogsource -n openshift-marketplace | grep self-node-remediation
```

#### Step 6: Delete SNR Deployments and DaemonSets

```bash
# List deployments
oc get deployment -n openshift-workload-availability | grep self-node-remediation

# Delete controller manager deployment
oc delete deployment self-node-remediation-controller-manager -n openshift-workload-availability

# List DaemonSets
oc get daemonset -n openshift-workload-availability | grep self-node-remediation

# Delete DaemonSet
oc delete daemonset self-node-remediation-ds -n openshift-workload-availability

# Verify deletion
oc get pods -n openshift-workload-availability | grep self-node-remediation
```

#### Step 7: Delete SNR Services

```bash
# List services
oc get service -n openshift-workload-availability | grep self-node-remediation

# Delete services
oc delete service self-node-remediation-controller-manager-metrics-service -n openshift-workload-availability
oc delete service self-node-remediation-webhook-service -n openshift-workload-availability

# Verify deletion
oc get service -n openshift-workload-availability | grep self-node-remediation
```

#### Step 8: Delete SNR RBAC

```bash
# Delete ServiceAccount
oc delete serviceaccount self-node-remediation-controller-manager -n openshift-workload-availability

# Delete RoleBindings
oc delete rolebinding -l app.kubernetes.io/name=self-node-remediation -n openshift-workload-availability

# Delete ClusterRoleBindings
oc get clusterrolebinding | grep self-node-remediation
oc delete clusterrolebinding self-node-remediation-controller-manager
oc delete clusterrolebinding self-node-remediation-manager-rolebinding

# Delete ClusterRoles
oc get clusterrole | grep self-node-remediation
oc delete clusterrole self-node-remediation-controller-manager
oc delete clusterrole self-node-remediation-manager-role
```

#### Step 9: Delete SNR ConfigMaps and Secrets

```bash
# List ConfigMaps
oc get configmap -n openshift-workload-availability | grep self-node-remediation

# Delete ConfigMaps
oc delete configmap -l app.kubernetes.io/name=self-node-remediation -n openshift-workload-availability

# List Secrets
oc get secret -n openshift-workload-availability | grep self-node-remediation

# Delete Secrets
oc delete secret -l app.kubernetes.io/name=self-node-remediation -n openshift-workload-availability
```

---

### PART 2: Node Health Check Operator

#### Step 1: Delete NHC Custom Resources

```bash
# List all NodeHealthCheck resources
oc get nodehealthcheck -A

# Delete all NodeHealthCheck resources
oc delete nodehealthcheck --all

# Verify deletion
oc get nodehealthcheck -A
```

#### Step 2: Delete NHC Subscription

```bash
# List subscriptions
oc get subscription -n openshift-workload-availability
oc get subscription -n openshift-workload-availability

# Delete NHC subscription
oc delete subscription node-healthcheck-operator -n openshift-workload-availability

# Verify deletion
oc get subscription -n openshift-workload-availability | grep node-healthcheck
```

#### Step 3: Delete NHC CSV

```bash
# List CSVs
oc get csv -n openshift-workload-availability

# Delete NHC CSV (adjust version as needed)
oc delete csv node-healthcheck-operator.v0.8.0 -n openshift-workload-availability

# Or delete by label
oc delete csv -l operators.coreos.com/node-healthcheck-operator.openshift-workload-availability \
    -n openshift-workload-availability

# Verify deletion
oc get csv -n openshift-workload-availability | grep node-healthcheck
```

#### Step 4: Delete NHC InstallPlan

```bash
# List InstallPlans
oc get installplan -n openshift-workload-availability

# Delete NHC InstallPlans
oc delete installplan -l operators.coreos.com/node-healthcheck-operator.openshift-workload-availability \
    -n openshift-workload-availability

# Verify deletion
oc get installplan -n openshift-workload-availability
```

#### Step 5: Delete NHC CatalogSource (if custom)

```bash
# List CatalogSources
oc get catalogsource -n openshift-marketplace

# Delete NHC CatalogSource (if you created a custom one)
oc delete catalogsource node-healthcheck-cs -n openshift-marketplace

# Note: If NHC was installed from community-operators or redhat-operators,
# don't delete those CatalogSources as they contain other operators

# Verify deletion
oc get catalogsource -n openshift-marketplace | grep node-healthcheck
```

#### Step 6: Delete NHC Deployments

```bash
# List deployments
oc get deployment -n openshift-workload-availability | grep node-healthcheck

# Delete controller manager deployment
oc delete deployment node-healthcheck-controller-manager -n openshift-workload-availability

# Verify deletion
oc get pods -n openshift-workload-availability | grep node-healthcheck
```

#### Step 7: Delete NHC Services

```bash
# List services
oc get service -n openshift-workload-availability | grep node-healthcheck

# Delete services
oc delete service node-healthcheck-controller-manager-metrics-service -n openshift-workload-availability
oc delete service node-healthcheck-webhook-service -n openshift-workload-availability

# Verify deletion
oc get service -n openshift-workload-availability | grep node-healthcheck
```

#### Step 8: Delete NHC RBAC

```bash
# Delete ServiceAccount
oc delete serviceaccount node-healthcheck-controller-manager -n openshift-workload-availability

# Delete RoleBindings
oc delete rolebinding -l app.kubernetes.io/name=node-healthcheck -n openshift-workload-availability

# Delete ClusterRoleBindings
oc get clusterrolebinding | grep node-healthcheck
oc delete clusterrolebinding node-healthcheck-controller-manager
oc delete clusterrolebinding node-healthcheck-manager-rolebinding

# Delete ClusterRoles
oc get clusterrole | grep node-healthcheck
oc delete clusterrole node-healthcheck-controller-manager
oc delete clusterrole node-healthcheck-manager-role
```

#### Step 9: Delete NHC ConfigMaps and Secrets

```bash
# List ConfigMaps
oc get configmap -n openshift-workload-availability | grep node-healthcheck

# Delete ConfigMaps
oc delete configmap -l app.kubernetes.io/name=node-healthcheck -n openshift-workload-availability

# List Secrets
oc get secret -n openshift-workload-availability | grep node-healthcheck

# Delete Secrets
oc delete secret -l app.kubernetes.io/name=node-healthcheck -n openshift-workload-availability
```

---

### PART 3: Delete CRDs (Optional - Cluster-wide Impact)

**⚠️ WARNING**: Deleting CRDs will remove them cluster-wide and delete all instances of these resources across all namespaces.

#### SNR CRDs

```bash
# List SNR CRDs
oc get crd | grep self-node-remediation

# Delete SNR CRDs (only if you're sure)
oc delete crd selfnoderemediationconfigs.self-node-remediation.medik8s.io
oc delete crd selfnoderemediations.self-node-remediation.medik8s.io
oc delete crd selfnoderemediationtemplates.self-node-remediation.medik8s.io

# Verify deletion
oc get crd | grep self-node-remediation
```

#### NHC CRDs

```bash
# List NHC CRDs
oc get crd | grep nodehealthcheck

# Delete NHC CRDs (only if you're sure)
oc delete crd nodehealthchecks.remediation.medik8s.io

# Verify deletion
oc get crd | grep nodehealthcheck
```

---

### PART 4: Delete OperatorGroup (Optional)

**⚠️ WARNING**: Only delete if no other operators are using it.

```bash
# List OperatorGroups
oc get operatorgroup -n openshift-workload-availability

# Check if other operators are using it
oc get subscription -n openshift-workload-availability

# If no other operators, you can delete
# oc delete operatorgroup workload-availability-operator-group -n openshift-workload-availability
```

---

## Verification

### Comprehensive Verification Script

```bash
#!/bin/bash

echo "=== Verification ==="
echo ""

echo "=========================================="
echo "SNR Operator Verification"
echo "=========================================="

echo "1. Checking SNR CSV..."
oc get csv -n openshift-workload-availability | grep self-node-remediation || echo "✓ No SNR CSV found"

echo "2. Checking SNR Subscription..."
oc get subscription -n openshift-workload-availability | grep self-node-remediation || echo "✓ No SNR Subscription found"

echo "3. Checking SNR Pods..."
oc get pods -n openshift-workload-availability | grep self-node-remediation || echo "✓ No SNR pods found"

echo "4. Checking SNR Deployments..."
oc get deployment -n openshift-workload-availability | grep self-node-remediation || echo "✓ No SNR deployments found"

echo "5. Checking SNR DaemonSets..."
oc get daemonset -n openshift-workload-availability | grep self-node-remediation || echo "✓ No SNR DaemonSets found"

echo "6. Checking SNR CatalogSource..."
oc get catalogsource -n openshift-marketplace | grep self-node-remediation || echo "✓ No SNR CatalogSource found"

echo "7. Checking SNR CRDs..."
oc get crd | grep self-node-remediation || echo "✓ No SNR CRDs found"

echo "8. Checking SNR Custom Resources..."
oc get selfnoderemediation -A 2>/dev/null || echo "✓ No SelfNodeRemediation resources found"
oc get selfnoderemediationconfig -A 2>/dev/null || echo "✓ No SelfNodeRemediationConfig resources found"
oc get selfnoderemediationtemplate -A 2>/dev/null || echo "✓ No SelfNodeRemediationTemplate resources found"

echo ""
echo "=========================================="
echo "NHC Operator Verification"
echo "=========================================="

echo "1. Checking NHC CSV..."
oc get csv -n openshift-workload-availability | grep node-healthcheck || echo "✓ No NHC CSV found"

echo "2. Checking NHC Subscription..."
oc get subscription -n openshift-workload-availability | grep node-healthcheck || echo "✓ No NHC Subscription found"

echo "3. Checking NHC Pods..."
oc get pods -n openshift-workload-availability | grep node-healthcheck || echo "✓ No NHC pods found"

echo "4. Checking NHC Deployments..."
oc get deployment -n openshift-workload-availability | grep node-healthcheck || echo "✓ No NHC deployments found"

echo "5. Checking NHC CatalogSource..."
oc get catalogsource -n openshift-marketplace | grep node-healthcheck || echo "✓ No NHC CatalogSource found"

echo "6. Checking NHC CRDs..."
oc get crd | grep nodehealthcheck || echo "✓ No NHC CRDs found"

echo "7. Checking NHC Custom Resources..."
oc get nodehealthcheck -A 2>/dev/null || echo "✓ No NodeHealthCheck resources found"

echo ""
echo "=========================================="
echo "Namespace Status"
echo "=========================================="

echo "Remaining resources in namespace:"
oc get all -n openshift-workload-availability

echo ""
echo "=== Verification Complete ==="
```

### Quick Verification Commands

```bash
# SNR verification
oc get csv,subscription,installplan -n openshift-workload-availability | grep self-node-remediation
oc get pods,deployment,daemonset -n openshift-workload-availability | grep self-node-remediation
oc get catalogsource -n openshift-marketplace | grep self-node-remediation
oc get crd | grep self-node-remediation

# NHC verification
oc get csv,subscription,installplan -n openshift-workload-availability | grep node-healthcheck
oc get pods,deployment -n openshift-workload-availability | grep node-healthcheck
oc get catalogsource -n openshift-marketplace | grep node-healthcheck
oc get crd | grep nodehealthcheck

# Check all resources in namespace
oc get all -n openshift-workload-availability
```

---

## Cleanup Local Images

### Remove SNR Local Container Images

```bash
# List SNR images
podman images | grep self-node-remediation

# Remove operator image
podman rmi quay.io/<your-username>/self-node-remediation-operator:s390x

# Remove bundle image
podman rmi quay.io/<your-username>/self-node-remediation-operator-bundle:s390x

# Remove catalog image
podman rmi quay.io/<your-username>/self-node-remediation-operator-catalog:s390x

# Verify removal
podman images | grep self-node-remediation
```

### Remove NHC Local Container Images (if built locally)

```bash
# List NHC images
podman images | grep node-healthcheck

# Remove operator image
podman rmi quay.io/<your-username>/node-healthcheck-operator:s390x

# Remove bundle image
podman rmi quay.io/<your-username>/node-healthcheck-operator-bundle:s390x

# Remove catalog image
podman rmi quay.io/<your-username>/node-healthcheck-operator-catalog:s390x

# Verify removal
podman images | grep node-healthcheck
```

### Remove Images from Quay.io (Optional)

#### Using Quay.io Web Interface

1. Go to https://quay.io
2. Navigate to your repositories
3. Delete the repositories:
   - SNR: `self-node-remediation-operator`, `self-node-remediation-operator-bundle`, `self-node-remediation-operator-catalog`
   - NHC: `node-healthcheck-operator`, `node-healthcheck-operator-bundle`, `node-healthcheck-operator-catalog`

#### Using Quay.io API

```bash
# Set variables
QUAY_USER="<your-username>"
QUAY_TOKEN="<your-token>"

# Delete SNR repositories
curl -X DELETE \
  -H "Authorization: Bearer ${QUAY_TOKEN}" \
  "https://quay.io/api/v1/repository/${QUAY_USER}/self-node-remediation-operator"

curl -X DELETE \
  -H "Authorization: Bearer ${QUAY_TOKEN}" \
  "https://quay.io/api/v1/repository/${QUAY_USER}/self-node-remediation-operator-bundle"

curl -X DELETE \
  -H "Authorization: Bearer ${QUAY_TOKEN}" \
  "https://quay.io/api/v1/repository/${QUAY_USER}/self-node-remediation-operator-catalog"

# Delete NHC repositories (if you created them)
curl -X DELETE \
  -H "Authorization: Bearer ${QUAY_TOKEN}" \
  "https://quay.io/api/v1/repository/${QUAY_USER}/node-healthcheck-operator"

curl -X DELETE \
  -H "Authorization: Bearer ${QUAY_TOKEN}" \
  "https://quay.io/api/v1/repository/${QUAY_USER}/node-healthcheck-operator-bundle"

curl -X DELETE \
  -H "Authorization: Bearer ${QUAY_TOKEN}" \
  "https://quay.io/api/v1/repository/${QUAY_USER}/node-healthcheck-operator-catalog"
```

---

## Troubleshooting

### Issue 1: CSV Won't Delete

**Symptom**: CSV stuck in "Deleting" state

**Solution**:
```bash
# For SNR
oc patch csv self-node-remediation.v0.0.1 -n openshift-workload-availability \
    -p '{"metadata":{"finalizers":[]}}' --type=merge
oc delete csv self-node-remediation.v0.0.1 -n openshift-workload-availability --force --grace-period=0

# For NHC
oc patch csv node-healthcheck-operator.v0.8.0 -n openshift-workload-availability \
    -p '{"metadata":{"finalizers":[]}}' --type=merge
oc delete csv node-healthcheck-operator.v0.8.0 -n openshift-workload-availability --force --grace-period=0
```

### Issue 2: Pods Won't Terminate

**Symptom**: Pods stuck in "Terminating" state

**Solution**:
```bash
# For SNR pods
oc delete pod -l app.kubernetes.io/name=self-node-remediation -n openshift-workload-availability --force --grace-period=0

# For NHC pods
oc delete pod -l app.kubernetes.io/name=node-healthcheck -n openshift-workload-availability --force --grace-period=0

# Force delete specific pod
oc delete pod <pod-name> -n openshift-workload-availability --force --grace-period=0
```

### Issue 3: CRD Won't Delete

**Symptom**: CRD stuck in "Terminating" state

**Solution**:
```bash
# For SNR CRDs
oc patch crd selfnoderemediations.self-node-remediation.medik8s.io \
    -p '{"metadata":{"finalizers":[]}}' --type=merge
oc delete crd selfnoderemediations.self-node-remediation.medik8s.io --force --grace-period=0

oc patch crd selfnoderemediationconfigs.self-node-remediation.medik8s.io \
    -p '{"metadata":{"finalizers":[]}}' --type=merge
oc delete crd selfnoderemediationconfigs.self-node-remediation.medik8s.io --force --grace-period=0

oc patch crd selfnoderemediationtemplates.self-node-remediation.medik8s.io \
    -p '{"metadata":{"finalizers":[]}}' --type=merge
oc delete crd selfnoderemediationtemplates.self-node-remediation.medik8s.io --force --grace-period=0

# For NHC CRDs
oc patch crd nodehealthchecks.remediation.medik8s.io \
    -p '{"metadata":{"finalizers":[]}}' --type=merge
oc delete crd nodehealthchecks.remediation.medik8s.io --force --grace-period=0
```

### Issue 4: Custom Resources Won't Delete

**Symptom**: Custom resources stuck in "Terminating"

**Solution**:
```bash
# For SNR resources
oc get selfnoderemediation -A
oc patch selfnoderemediation <resource-name> -n <namespace> \
    -p '{"metadata":{"finalizers":[]}}' --type=merge
oc delete selfnoderemediation <resource-name> -n <namespace> --force --grace-period=0

# For NHC resources
oc get nodehealthcheck -A
oc patch nodehealthcheck <resource-name> \
    -p '{"metadata":{"finalizers":[]}}' --type=merge
oc delete nodehealthcheck <resource-name> --force --grace-period=0
```

### Issue 5: Namespace Won't Delete

**Symptom**: Namespace stuck in "Terminating" (if you're trying to delete the namespace)

**Solution**:
```bash
# Check what's preventing deletion
oc api-resources --verbs=list --namespaced -o name | \
    xargs -n 1 oc get --show-kind --ignore-not-found -n openshift-workload-availability

# Remove finalizers from namespace
oc get namespace openshift-workload-availability -o json | \
    jq '.spec.finalizers = []' | \
    oc replace --raw "/api/v1/namespaces/openshift-workload-availability/finalize" -f -
```

### Issue 6: CatalogSource Pod Keeps Restarting

**Symptom**: After deleting CatalogSource, pod keeps restarting

**Solution**:
```bash
# Delete the pod forcefully
oc get pods -n openshift-marketplace | grep self-node-remediation
oc delete pod <catalogsource-pod-name> -n openshift-marketplace --force --grace-period=0

# Verify CatalogSource is deleted
oc get catalogsource -n openshift-marketplace | grep self-node-remediation
```

### Issue 7: Webhook Preventing Resource Deletion

**Symptom**: Error about webhook when trying to delete resources

**Solution**:
```bash
# Delete webhook configurations
oc delete validatingwebhookconfiguration self-node-remediation-validating-webhook-configuration
oc delete mutatingwebhookconfiguration self-node-remediation-mutating-webhook-configuration

oc delete validatingwebhookconfiguration node-healthcheck-validating-webhook-configuration
oc delete mutatingwebhookconfiguration node-healthcheck-mutating-webhook-configuration

# Then retry deletion
```

---

## Complete Cleanup Checklist

Use this checklist to ensure complete removal:

### SNR Operator
- [ ] SelfNodeRemediation CRs deleted
- [ ] SelfNodeRemediationConfig CRs deleted
- [ ] SelfNodeRemediationTemplate CRs deleted
- [ ] SNR Subscription deleted
- [ ] SNR CSV deleted
- [ ] SNR InstallPlan deleted
- [ ] SNR CatalogSource deleted
- [ ] SNR Deployments deleted
- [ ] SNR DaemonSets deleted
- [ ] SNR Services deleted
- [ ] SNR ServiceAccount deleted
- [ ] SNR RoleBindings deleted
- [ ] SNR ClusterRoleBindings deleted
- [ ] SNR ClusterRoles deleted
- [ ] SNR ConfigMaps deleted
- [ ] SNR Secrets deleted
- [ ] SNR CRDs deleted (optional)
- [ ] SNR local images removed
- [ ] SNR Quay.io images removed (optional)

### NHC Operator
- [ ] NodeHealthCheck CRs deleted
- [ ] NHC Subscription deleted
- [ ] NHC CSV deleted
- [ ] NHC InstallPlan deleted
- [ ] NHC CatalogSource deleted (if custom)
- [ ] NHC Deployments deleted
- [ ] NHC Services deleted
- [ ] NHC ServiceAccount deleted
- [ ] NHC RoleBindings deleted
- [ ] NHC ClusterRoleBindings deleted
- [ ] NHC ClusterRoles deleted
- [ ] NHC ConfigMaps deleted
- [ ] NHC Secrets deleted
- [ ] NHC CRDs deleted (optional)
- [ ] NHC local images removed (if built)
- [ ] NHC Quay.io images removed (optional)

### Common
- [ ] OperatorGroup deleted (optional, if not used by others)
- [ ] Namespace deleted (optional, if no other resources)

---

## Summary

### What Gets Deleted

#### SNR Operator - Namespace-scoped Resources (in `openshift-workload-availability`)
- Subscription
- CSV (ClusterServiceVersion)
- InstallPlan
- Deployment (controller-manager)
- DaemonSet (self-node-remediation-ds)
- Services
- ServiceAccount
- RoleBindings
- ConfigMaps
- Secrets
- Custom Resources (SelfNodeRemediation, Config, Template)

#### SNR Operator - Cluster-scoped Resources
- CatalogSource (in `openshift-marketplace`)
- ClusterRoleBindings
- ClusterRoles
- CRDs (optional)

#### NHC Operator - Namespace-scoped Resources (in `openshift-workload-availability`)
- Subscription
- CSV (ClusterServiceVersion)
- InstallPlan
- Deployment (controller-manager)
- Services
- ServiceAccount
- RoleBindings
- ConfigMaps
- Secrets
- Custom Resources (NodeHealthCheck)

#### NHC Operator - Cluster-scoped Resources
- CatalogSource (in `openshift-marketplace`, if custom)
- ClusterRoleBindings
- ClusterRoles
- CRDs (optional)

#### Local Resources
- Container images (operator, bundle, catalog for both SNR and NHC)

#### Remote Resources (optional)
- Quay.io repositories

### Final Verification

```bash
# Complete verification - should return no results
echo "=== Final Verification ==="

# SNR
oc get csv,subscription,installplan,deployment,daemonset,pod -n openshift-workload-availability | grep self-node-remediation
oc get catalogsource -n openshift-marketplace | grep self-node-remediation
oc get crd | grep self-node-remediation
oc get selfnoderemediation,selfnoderemediationconfig,selfnoderemediationtemplate -A

# NHC
oc get csv,subscription,installplan,deployment,pod -n openshift-workload-availability | grep node-healthcheck
oc get catalogsource -n openshift-marketplace | grep node-healthcheck
oc get crd | grep nodehealthcheck
oc get nodehealthcheck -A

echo "If all commands return no results, both operators are completely removed."
```

---

**Document Version**: 1.0  
**Last Updated**: 2026-04-14  
**Purpose**: Complete uninstallation of Self Node Remediation and Node Health Check operators