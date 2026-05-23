# Kubernetes Schedulers
**Class Notes** | **Date:** 18/05/2026

## Overview: Scheduling Methods

The Kubernetes Scheduler assigns pods to nodes. It uses the following methods:
- **DaemonSet**
- **Node Selector**
- **Node Affinity**
  - Preferred — Soft rule (ignored during execution)
  - Required — Hard rule (ignored during execution)
- **Taint and Toleration**

---

## 1. DaemonSet

A DaemonSet ensures that exactly one Pod runs on every node in the cluster. It is used for background agents like log collectors, monitoring exporters, or network plugins.

### Key Points
- Schedules one Pod on each node automatically
- Used to deploy monitoring agents or node exporters into every node
- **Desired State = Node Count** — pod count always equals node count

### Behavior
- Delete a pod → It is recreated immediately on the same node
- Add a new node → DaemonSet automatically creates a pod on that node
- You cannot change instance type when server is in Auto Scaling Group Control
- Use DaemonSet whenever you need something running on each node

> **Note:** DaemonSet = Desired State. Pod count always matches node count.

### Lab Commands
```bash
# Get all nodes
kubectl get nodes

# Deploy DaemonSet
kubectl apply -f daemonset.yml

# Verify pods
kubectl get pods

# Delete DaemonSet
kubectl delete -f daemonset.yml
```

---

## 2. Node Selector

Node Selector is the simplest scheduling constraint. You label a node, then reference that label in your pod spec. The pod will only run on nodes that match the label.

### How It Works
- First label the node with a key=value pair
- Add the same label as nodeSelector in your pod/deployment YAML
- Pod will be scheduled only on the matching node
- If labels mismatch or node is unlabeled → pod stays in Pending status
- Labeled pod **CANNOT** schedule on an unlabeled node
- Labeled node **IS** allowed to schedule unlabeled pods — Yes

> **Note:** If labels mismatch or node is unlabeled → pod will go to Pending state.

### Lab Commands
```bash
# Apply node selector YAML
kubectl apply -f node-selector.yml

# Check pods
kubectl get pods

# Get nodes
kubectl get nodes

# Label a node
kubectl label nodes <node-name> clr=white

# Show labels
kubectl get nodes --show-labels

# Wide view
kubectl get pods -o wide

# Unlabel a node
kubectl label nodes <node-name> clr-
```

---

## 3. Node Affinity

Node Affinity is an advanced version of Node Selector. It allows more expressive rules and supports logical operators (In, NotIn, Exists, etc.).

### Types

**Preferred** (`preferredDuringSchedulingIgnoredDuringExecution`)
- Soft rule. Scheduler tries to place pod on matching node
- If no match found, pod is placed on any available node
- Ignored during execution — running pods are not evicted if labels change

**Required** (`requiredDuringSchedulingIgnoredDuringExecution`)
- Hard rule. Pod **MUST** go on a matching node
- If no match found, pod stays in Pending state
- Ignored during execution — same as Preferred

### Difference from Node Selector
- Supports additional expressions: In, NotIn, Exists, DoesNotExist, Gt, Lt
- More flexible than basic Node Selector
- Same concept, just more powerful syntax

> **Note:** Node Affinity is applied at scheduling time. Once pod is running, affinity rules are ignored.

### Commands
```bash
# Get pods
kubectl get pods

# Cordon a node (block new pods without evicting existing)
kubectl cordon <node-name>

# Wide output
kubectl get pods -o wide
```

---

## 4. Taint & Toleration

Taints are applied to nodes to repel pods. Tolerations are applied to pods to allow them to schedule on tainted nodes. Together they control which pods can go where.

### Concept Flow
- **Taint a Node** → Repels pods that don't tolerate the taint
- **Toleration on Pod** → Pod can tolerate the taint and get scheduled on the node
- **Untolerated pods** → Only deploy on Untainted nodes
- **Tolerated pods** → Can schedule on BOTH tainted and untainted nodes

### Taint Effects
- **NoSchedule** → No new pods will be scheduled on the node. Already running pods are unaffected
- **NoExecute** → No new pods scheduled AND existing untolerated pods are evicted from the node

### Taint Node vs Cordon Node
- **Taint Node:** Only tolerating pods can be scheduled → Selective control
- **Cordon Node:** Absolutely no new pods scheduled → Full block (for maintenance)

### Scenario: Pods Running on Untainted Node — Taint Applied After

**Effect = NoSchedule:**
- No impact on currently running/scheduled pods
- New pods without toleration won't schedule

**Effect = NoExecute:**
- Untolerated pods → Go to Pending, get deleted, Kubernetes reschedules on another available node
- Tolerated pods → Continue running on the tainted node without disruption

### Pod in Pending Status — Possible Reasons
- Labels mismatch or node is unlabeled (Node Selector issue)
- Required Node Affinity rule not satisfied
- Node has a taint and pod has no matching toleration
- Insufficient CPU/memory resources on available nodes

### Commands
```bash
# Taint a node
kubectl taint nodes <node-name> key=value:NoSchedule

# Remove taint
kubectl taint nodes <node-name> key=value:NoSchedule-

# Check pods
kubectl get pods

# Wide output
kubectl get pods -o wide
```

---

**— End of Notes —**
