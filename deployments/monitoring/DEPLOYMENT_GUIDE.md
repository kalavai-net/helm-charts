# Production Monitoring Stack Deployment Guide

## Required Information Checklist

Gather all the values before proceeding with deployment.

---

## 1. Kubernetes Cluster

| Item | Value |
|------|-------|
| Cluster context name | `__________________` |

Get age private encryption key.

**Verify access:**
```bash
kubectl config get-contexts
kubectl --context=YOUR_CONTEXT get nodes
kubectl --context=YOUR_CONTEXT auth can-i create namespace
```

---

## 2. Monitoring Nodes (5 Required)

All 5 nodes must be in central datacenter location.

| Node # | Node Name | Created | Labeled | Tainted |
|--------|-----------|---------|---------|---------|
| 1 | `________________________` | [ ] | [ ] | [ ] |
| 2 | `________________________` | [ ] | [ ] | [ ] |
| 3 | `________________________` | [ ] | [ ] | [ ] |
| 4 | `________________________` | [ ] | [ ] | [ ] |
| 5 | `________________________` | [ ] | [ ] | [ ] |

**Minimum node specs:**
- 4 CPU cores
- 16GB RAM
- Network connectivity to control plane

**Label command (run for each node):**
```bash
kubectl label node NODE_NAME monitoring-master=true
```

**Taint command (run for each node):**
```bash
kubectl taint node NODE_NAME monitoring-workload=central:NoSchedule
```

**Verification:**
```bash
# Verify labels
kubectl get nodes -l monitoring-master=true
# Should show 5 nodes

# Verify taints
kubectl describe node NODE_NAME | grep Taint
# Should show: monitoring-workload=central:NoSchedule
```

---

## 3. S3 Storage (Scaleway)

| Item | Value |
|------|-------|
| Endpoint | `s3.fr-par.scw.cloud` |
| Access Key ID | `__________________` |
| Secret Access Key | `__________________` |

**Required Buckets (must exist):**

| Bucket Name | Purpose | Exists |
|-------------|---------|--------|
| `monitoring-mimir-blocks` | Metrics storage | [ ] |
| `monitoring-mimir-alertmanager` | Alertmanager state | [ ] |
| `monitoring-mimir-ruler` | Recording/alerting rules | [ ] |

**Verify bucket access:**
```bash
# Using s3cmd or aws cli configured for Scaleway
aws --endpoint-url=https://s3.fr-par.scw.cloud s3 ls s3://monitoring-mimir-blocks/
```

---

## 4. Grafana Credentials

| Item | Value |
|------|-------|
| Admin username | `admin` (recommended) |
| Admin password | `__________________` |

**Password requirements:**
- Minimum 12 characters
- Mix of upper/lowercase, numbers, special chars

---

## 5. Projects Configuration

Projects with nodes labeled `geo=project-name`:

| Project Name | Nodes Labeled `geo=X` |
|--------------|----------------------|
| shadow-camtl01 | [ ] Verified |
| shadow-frsbg01 | [ ] Verified |

**Verify project labels on cluster nodes:**
```bash
kubectl get nodes -l geo=shadow-camtl01
kubectl get nodes -l geo=shadow-frsbg01
```

---

## Deployment Steps

### Step 1: Verify Infrastructure

```bash
# Set context
export KUBE_CONTEXT=your-prod-context-name

# Check nodes ready
kubectl --context=$KUBE_CONTEXT get nodes -l monitoring-master=true

# Check taints applied
for node in NODE1 NODE2 NODE3 NODE4 NODE5; do
  echo "=== $node ==="
  kubectl --context=$KUBE_CONTEXT describe node $node | grep Taint
done
```

### Step 2: Update Secrets File

Decrypt file with production secrets.

```bash
sops -i -e env/prod/secrets.yaml
```


Edit `env/prod/secrets.yaml` with actual values:

```yaml
grafana_admin_username: admin
grafana_admin_password: YOUR_ACTUAL_PASSWORD
s3:
  access_key: YOUR_SCALEWAY_ACCESS_KEY
  secret_key: YOUR_SCALEWAY_SECRET_KEY
```

### Step 3: Encrypt Secrets

```bash
sops -i -e env/prod/secrets.yaml

# Verify encryption
head -20 env/prod/secrets.yaml
# Should show encrypted content
```

### Step 4: Deploy

Only for initial deployment run:

```bash
cd helm-charts/deployments/monitoring
export KUBE_CONTEXT=your-prod-context-name
helmfile -e prod apply --skip-diff-on-install
```
For subsequent deployments run:

```bash
cd helm-charts/deployments/monitoring
export KUBE_CONTEXT=your-prod-context-name
helmfile -e prod apply
```

### Step 6: Monitor Deployment

```bash
# Watch pods
kubectl --context=$KUBE_CONTEXT get pods -n mon -w

# Check pod distribution
kubectl --context=$KUBE_CONTEXT get pods -n mon -o wide
```

### Step 7: Verify Deployment

```bash
# All pods running
kubectl --context=$KUBE_CONTEXT get pods -n mon

# Prometheus instances
kubectl --context=$KUBE_CONTEXT get prometheus -n mon

# ServiceMonitors
kubectl --context=$KUBE_CONTEXT get servicemonitor -n mon

# Mimir health
kubectl --context=$KUBE_CONTEXT logs -n mon deployment/mon-mimir-distributor --tail=20
```
Once deployment is deployed, you will see multiple instances of prometheus-node-exported. List them,
and label them with geo label which corresponds to geo label of the node the instance is running on

```bash
kubectl get po -n mon -l app.kubernetes.io/name=prometheus-node-exporter

# Label found pods
kubectl label pod <pod name> geo=<project name>
```
At this moment this step is manaual, will be replaced with automation provided by autoscaling operator.


### Step 8: Access Grafana

```bash
# Get credentials
kubectl --context=$KUBE_CONTEXT get secret -n mon mon-grafana \
  -o jsonpath='{.data.admin-password}' | base64 -d && echo

# Port-forward
kubectl --context=$KUBE_CONTEXT port-forward -n mon svc/mon-grafana 3000:80

# Open browser: http://localhost:3000
```

### Step 9: Verify Metrics Flow

In Grafana Explore, test queries:
```promql
up{job="kubelet"}
up{job=~".*node-exporter.*"}
kube_pod_info
```

---

## Troubleshooting

### Pods Pending - Toleration Issues
```bash
kubectl --context=$KUBE_CONTEXT describe pod -n mon POD_NAME | grep -A10 Events
kubectl --context=$KUBE_CONTEXT get nodes -l monitoring-master=true --show-labels
```

### S3 Connection Errors
```bash
kubectl --context=$KUBE_CONTEXT logs -n mon deployment/mon-mimir-distributor | grep -i error
kubectl --context=$KUBE_CONTEXT get secret -n mon mimir-s3-credentials -o yaml
```

### Prometheus Not Scraping
```bash
kubectl --context=$KUBE_CONTEXT port-forward -n mon prometheus-prometheus-central-0 9090:9090
# Check http://localhost:9090/targets
```

