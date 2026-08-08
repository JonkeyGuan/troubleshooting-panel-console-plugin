# Troubleshooting Panel + Lightspeed Setup on COO-Managed Cluster

> Tested on OCP 4.22.8 with COO (Cluster Observability Operator) pre-installed, Lightspeed Operator v1.1.2, and Korrel8r v0.11.6

## Prerequisites

- OpenShift cluster 4.22+ with COO installed
- UIPlugin `troubleshooting-panel` already created by COO
- `oc` CLI installed, logged in as cluster-admin
- LLM API key with function calling support (MiniMax-M3 or compatible)

## 1. Upgrade Korrel8r to v0.11.6

COO deploys Korrel8r v0.11.1 which lacks MCP support. Upgrade to v0.11.6:

```bash
# Scale down COO to prevent it from reverting changes
oc scale deployment observability-operator -n openshift-cluster-observability-operator --replicas=0

# Upgrade korrel8r image
oc set image deployment/korrel8r -n openshift-cluster-observability-operator \
  korrel8r=quay.io/korrel8r/korrel8r:0.11.6

# Wait for rollout
oc rollout status deployment/korrel8r -n openshift-cluster-observability-operator --timeout=60s
```

### 1.1 Grant tokenreviews Permission

Korrel8r v0.11.6 requires `tokenreviews` access for MCP authentication:

```bash
oc apply -f - <<'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: korrel8r-token-review
rules:
- apiGroups: ["authentication.k8s.io"]
  resources: ["tokenreviews"]
  verbs: ["create"]
EOF

oc create clusterrolebinding korrel8r-token-review-binding \
  --clusterrole=korrel8r-token-review \
  --serviceaccount=openshift-cluster-observability-operator:troubleshooting-panel-sa
```

### 1.2 Create External Route

```bash
oc create route reencrypt korrel8r --service=korrel8r -n openshift-cluster-observability-operator
```

Verify:

```bash
KORREL8R_URL=$(oc get routes/korrel8r -n openshift-cluster-observability-operator -o template='https://{{.spec.host}}')
TOKEN=$(oc whoami -t)
curl -sk "$KORREL8R_URL/api/v1alpha1/domains" -H "Authorization: Bearer $TOKEN" | python3 -m json.tool
```

## 2. Enable Agent Navigation

Update the UIPlugin CR to enable agent navigation:

```bash
oc patch uiplugin troubleshooting-panel --type=merge -p '{
  "spec": {
    "troubleshootingPanel": {
      "enableAgentNavigation": true
    }
  }
}'
```

> **Note:** COO must be scaled back up briefly to reconcile this change, then scaled down again to preserve the korrel8r upgrade.

```bash
# Scale COO up to reconcile
oc scale deployment observability-operator -n openshift-cluster-observability-operator --replicas=1

# Wait for reconciliation (~30s)
sleep 30
oc rollout status deployment/troubleshooting-panel -n openshift-cluster-observability-operator --timeout=60s

# Scale COO back down
oc scale deployment observability-operator -n openshift-cluster-observability-operator --replicas=0

# Re-upgrade korrel8r (COO reverted it)
oc set image deployment/korrel8r -n openshift-cluster-observability-operator \
  korrel8r=quay.io/korrel8r/korrel8r:0.11.6
oc rollout status deployment/korrel8r -n openshift-cluster-observability-operator --timeout=60s
```

### 2.1 Re-enable Console Plugin

COO reconciliation may reset the console plugins list. Re-add if missing:

```bash
# Check current plugins
oc get consoles.operator.openshift.io cluster -o jsonpath='{.spec.plugins}'

# Add if troubleshooting-panel-console-plugin is not listed
oc patch consoles.operator.openshift.io cluster --type=json \
  -p='[{"op":"add","path":"/spec/plugins/-","value":"troubleshooting-panel-console-plugin"}]'
```

## 3. Install OpenShift Lightspeed

### 3.1 Install Lightspeed Operator

```bash
oc apply -f - <<'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-lightspeed
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: openshift-lightspeed
  namespace: openshift-lightspeed
spec:
  targetNamespaces:
  - openshift-lightspeed
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: lightspeed-operator
  namespace: openshift-lightspeed
spec:
  channel: stable
  name: lightspeed-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  installPlanApproval: Automatic
EOF
```

Wait for the operator to be ready:

```bash
oc wait csv -n openshift-lightspeed \
  -l operators.coreos.com/lightspeed-operator.openshift-lightspeed \
  --for=jsonpath='{.status.phase}'=Succeeded --timeout=300s
```

### 3.2 Create LLM API Key Secret

```bash
oc create secret generic minimax-api-key \
  --from-literal=apitoken=<YOUR_MINIMAX_API_KEY> \
  -n openshift-lightspeed
```

### 3.3 Configure OLSConfig

```bash
KORREL8R_URL=$(oc get routes/korrel8r -n openshift-cluster-observability-operator -o template='https://{{.spec.host}}')

oc apply -f - <<EOF
apiVersion: ols.openshift.io/v1alpha1
kind: OLSConfig
metadata:
  name: cluster
spec:
  featureGates:
  - MCPServer
  llm:
    providers:
    - name: minimax
      type: openai
      url: https://api.minimax.io/v1
      credentialsSecretRef:
        name: minimax-api-key
      models:
      - name: MiniMax-M3
  ols:
    defaultModel: MiniMax-M3
    defaultProvider: minimax
    maxIterations: 10
    toolsApprovalConfig:
      approvalType: never
    deployment:
      replicas: 1
  mcpServers:
  - name: korrel8r
    url: ${KORREL8R_URL}/mcp
    timeout: 30
    headers:
    - name: Authorization
      valueFrom:
        type: kubernetes
EOF
```

> **maxIterations**: Default is 5, which is too low for complex prompts requiring multiple tool calls. Set to 10 to allow the full tool chain to complete.

Wait for Lightspeed to be ready:

```bash
oc wait olsconfig cluster --for=jsonpath='{.status.overallStatus}'=Ready --timeout=300s -n openshift-lightspeed
```

## 4. Create Test Workloads (Optional)

```bash
oc create namespace demo-troubleshoot

# CrashLoopBackOff pod
oc apply -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: failing-app
  namespace: demo-troubleshoot
spec:
  replicas: 2
  selector:
    matchLabels:
      app: failing-app
  template:
    metadata:
      labels:
        app: failing-app
    spec:
      containers:
      - name: crash-container
        image: registry.access.redhat.com/ubi9/ubi-minimal:latest
        command: ["/bin/sh", "-c"]
        args: ["echo 'ERROR: Database connection refused'; exit 1"]
        resources:
          limits:
            memory: "128Mi"
            cpu: "100m"
EOF

# OOMKill pod
oc apply -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: oom-app
  namespace: demo-troubleshoot
spec:
  replicas: 1
  selector:
    matchLabels:
      app: oom-app
  template:
    metadata:
      labels:
        app: oom-app
    spec:
      containers:
      - name: memory-hog
        image: registry.access.redhat.com/ubi9/ubi-minimal:latest
        command: ["/bin/sh", "-c"]
        args: ["head -c 256m /dev/urandom > /dev/null"]
        resources:
          limits:
            memory: "32Mi"
            cpu: "100m"
EOF
```

## 5. Verification

### 5.1 Verify Korrel8r MCP Tools

After making a query in Lightspeed, check logs:

```bash
oc logs deployment/lightspeed-app-server -n openshift-lightspeed --tail=50 | grep "Loaded.*tools"
```

Expected:

```
Loaded 24 tools from MCP server 'openshift'
Loaded 8 tools from MCP server 'korrel8r'
```

### 5.2 Verify Signal Correlation Panel

1. Open the OpenShift console
2. Click **Signal Correlation** in the app launcher (grid icon, top-right)
3. The troubleshooting panel should open

### 5.3 Verify Agent Navigation

1. In the troubleshooting panel toolbar, click the **AI icon**
2. Toggle **Agent Navigation ON**
3. Status should show connected (green)

### 5.4 Test with Lightspeed

Open the Lightspeed chat and try:

```
find error pod in demo-troubleshoot ns, and show correlated signals and show it in console
```

## Key Differences from Standalone Setup

| Aspect | Standalone (setup.md) | COO-Managed (this guide) |
|--------|----------------------|--------------------------|
| Korrel8r | Community operator + separate namespace | COO-managed in `openshift-cluster-observability-operator` |
| Troubleshooting Panel | Helm chart deployment | COO-managed via UIPlugin CR |
| ConsolePlugin proxy | Manually configured | Auto-configured by COO |
| TLS | Manual certs | COO manages serving certs |
| Image override | Direct | Must scale down COO first |
| Agent Navigation | Env var: `TROUBLESHOOTING_PANEL_CONSOLE_PLUGIN_FEATURES` | UIPlugin CR: `enableAgentNavigation: true` |

## Known Issues

| Issue | Root Cause | Workaround |
|-------|-----------|------------|
| COO reverts korrel8r image | COO reconciliation restores managed resources | Keep COO scaled to 0 |
| Console plugin disappears after COO reconcile | COO resets console plugins list | Re-add with `oc patch consoles.operator.openshift.io` |
| `agent-navigation.patch.json` warning in logs | Patch file not included in any image build | Cosmetic only, does not affect functionality |
| MiniMax-M3 raw XML tool calls | Model limitation for complex tool schemas | Retry the prompt, or use a model with better function calling |
| `create_neighbors_graph` failures | Model generates incorrect korrel8r query syntax | Retry; results are intermittent |

## Cleanup

```bash
# Remove test workloads
oc delete namespace demo-troubleshoot

# Remove Lightspeed
oc delete olsconfig cluster -n openshift-lightspeed
oc delete subscription lightspeed-operator -n openshift-lightspeed
oc delete namespace openshift-lightspeed

# Restore COO (reverts korrel8r to default version)
oc scale deployment observability-operator -n openshift-cluster-observability-operator --replicas=1

# Remove RBAC
oc delete clusterrolebinding korrel8r-token-review-binding
oc delete clusterrole korrel8r-token-review

# Remove route
oc delete route korrel8r -n openshift-cluster-observability-operator

# Disable agent navigation
oc patch uiplugin troubleshooting-panel --type=merge -p '{"spec":{"troubleshootingPanel":{"enableAgentNavigation":false}}}'
```
