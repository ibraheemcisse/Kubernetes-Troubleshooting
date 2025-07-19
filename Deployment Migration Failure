# Scenario 001: Deployment Migration Failure

**Platform**: KillerCoda  
**Problem Type**: Configuration Issue  
**Resolution Time**: 28 minutes  
**Complexity**: Beginner  
**Date**: [Today's Date]

## Problem Analysis

### Issue Description
A Deployment imported from another Kubernetes cluster has Pods stuck in Pending status, preventing the application from running in the new cluster environment.

### Symptoms Observed
- All Pods in Pending state
- Deployment exists but replicas not becoming ready
- No obvious resource constraints or scheduling issues

### Initial Investigation
```bash
kubectl get pods
# Output showed:
# NAME                                   READY   STATUS    RESTARTS   AGE
# management-frontend-xxx-xxx           0/1     Pending   0          Xm
```

## Investigation Process

### Step 1: Check Pod Status
```bash
kubectl get pods -o wide
kubectl describe pod <pod-name>
```
**Observation**: Pods stuck in Pending state with scheduling issues

### Step 2: Examine Deployment Configuration
```bash
kubectl describe deployment management-frontend
kubectl get deployment management-frontend -o yaml
```
**Key Finding**: Suspected node-specific configuration from source cluster

### Step 3: Edit Deployment Specification
```bash
kubectl edit deploy management-frontend
```
**Root Cause Identified**: Hard-coded `nodeName: staging-node1` in pod spec

## Solution Implementation

### Problem Root Cause
The Deployment contained a hard-coded `nodeName: staging-node1` field in the pod template, which referenced a node that doesn't exist in the target cluster.

### Resolution Steps
1. Edited the Deployment using `kubectl edit deploy management-frontend`
2. Removed the problematic `nodeName: staging-node1` line from the pod spec
3. Saved the configuration changes

### Fixed Configuration
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: management-frontend
spec:
  template:
    spec:
      containers:
      - image: nginx:alpine
        imagePullPolicy: IfNotPresent
        name: nginx
      dnsPolicy: ClusterFirst
      # nodeName: staging-node1 <-- REMOVED THIS LINE
```

### Verification
```bash
kubectl get pods
# All pods transitioned to Running state:
# NAME                                   READY   STATUS    RESTARTS   AGE
# management-frontend-6467b9fc4d-748pt   1/1     Running   0          21m
# management-frontend-6467b9fc4d-fl7t5   1/1     Running   0          22m
# management-frontend-6467b9fc4d-svspq   1/1     Running   0          22m
# management-frontend-6467b9fc4d-z2fnq   1/1     Running   0          22m
# management-frontend-6467b9fc4d-zd4x4   1/1     Running   0          21m
```

## Lessons Learned

### Technical Insights
- `nodeName` field forces pods to schedule on specific nodes
- When migrating between clusters, node-specific configurations cause scheduling failures
- Kubernetes scheduler cannot override hard-coded node assignments

### Best Practices Identified
- Use node selectors or affinity rules instead of hard-coded node names
- Always review manifests for environment-specific configurations before migration
- Test deployments in target environments before production migration

### Troubleshooting Methodology
- Start with `kubectl get pods` to identify status issues
- Use `kubectl describe` for detailed scheduling information
- Check deployment specs for hard-coded environment references
- Edit configurations to remove cluster-specific constraints

### Commands to Remember
```bash
kubectl get pods -o wide                    # Check pod placement and status
kubectl describe pod <name>                 # Detailed scheduling information
kubectl edit deploy <name>                  # Quick configuration fixes
kubectl get deployment <name> -o yaml       # Full configuration review
```

## Prevention Strategies

### For Production Environments
- Implement configuration validation pipelines
- Use tools like kubeval or OPA Gatekeeper to catch environment-specific configs
- Maintain separate manifests for different environments
- Document cluster migration procedures

---

**Key Takeaway**: Always sanitize manifests from environment-specific configurations when migrating between clusters. Hard-coded node names are a common migration pitfall.
