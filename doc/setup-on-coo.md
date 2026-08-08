# Troubleshooting Panel + Lightspeed Setup on COO-Managed Cluster

> Tested on OCP 4.22.8 with COO v1.5.1, Lightspeed Operator v1.1.2, Korrel8r Operator v0.1.8 (korrel8r v0.12.0)

## Prerequisites

- OpenShift cluster 4.22+ with COO installed
- Korrel8r Operator installed (v0.1.8+, provides korrel8r v0.12.0 with MCP support)
- `oc` CLI installed, logged in as cluster-admin
- Helm CLI installed
- LLM API key with function calling support (MiniMax-M3 or compatible)

## 1. Remove COO-Managed Troubleshooting Panel

COO deploys its own troubleshooting panel via a UIPlugin CR. We need to remove it first to avoid conflicts with our standalone deployment, which uses a custom image with the latest features (e.g. `useOverlay` for OCP 4.22).

```bash
# Delete the COO-managed UIPlugin (this removes COO's troubleshooting panel deployment)
oc delete uiplugin troubleshooting-panel

# Remove the console plugin registration if it was added
oc patch consoles.operator.openshift.io cluster --type=json \
  -p='[{"op":"remove","path":"/spec/plugins/0"}]' 2>/dev/null || true

# Verify it's gone
oc get uiplugin troubleshooting-panel 2>&1  # Should return NotFound
oc get deployment -A 2>&1 | grep troubleshoot  # Should return nothing
```

> **Note:** COO will not recreate the UIPlugin automatically. If you want to restore it later, re-create the UIPlugin CR with `spec.type: TroubleshootingPanel`.

## 2. Configure Korrel8r

The Korrel8r Operator (v0.1.8) deploys korrel8r v0.12.0 which includes MCP support. Install the operator from OperatorHub, then create a Korrel8r CR:

```bash
oc apply -f - <<'EOF'
apiVersion: korrel8r.openshift.io/v1alpha1
kind: Korrel8r
metadata:
  name: korrel8r
  namespace: korrel8r
spec: {}
EOF
```

### 2.1 Create External Route

```bash
oc create route reencrypt korrel8r --service=korrel8r -n korrel8r
```

Verify:

```bash
KORREL8R_URL=$(oc get routes/korrel8r -n korrel8r -o template='https://{{.spec.host}}')
TOKEN=$(oc whoami -t)
curl -sk "$KORREL8R_URL/api/v1alpha1/domains" -H "Authorization: Bearer $TOKEN" | python3 -m json.tool
```

## 3. Deploy Troubleshooting Panel (Standalone)

Deploy the troubleshooting panel standalone via Helm chart with a custom image:

```bash
# Deploy with Helm
helm upgrade --install troubleshooting-panel-console-plugin \
  charts/openshift-console-plugin \
  -n troubleshooting-panel-console-plugin --create-namespace \
  --set plugin.image=quay.io/jonkey/troubleshooting-panel-console-plugin:4.22
```

### 3.1 Enable Console Plugin

```bash
# Check current plugins
oc get consoles.operator.openshift.io cluster -o jsonpath='{.spec.plugins}'

# Add if troubleshooting-panel-console-plugin is not listed
oc patch consoles.operator.openshift.io cluster --type=json \
  -p='[{"op":"add","path":"/spec/plugins/-","value":"troubleshooting-panel-console-plugin"}]'
```

## 4. Install OpenShift Lightspeed

### 4.1 Install Lightspeed Operator

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

### 4.2 Create LLM API Key Secret

```bash
oc create secret generic minimax-api-key \
  --from-literal=apitoken=<YOUR_MINIMAX_API_KEY> \
  -n openshift-lightspeed
```

### 4.3 Build and Push RAG Knowledge Base (Optional)

Build a custom RAG knowledge base with korrel8r documentation to improve MCP tool call accuracy:

```bash
git clone https://github.com/JonkeyGuan/korrel8r-rag.git
cd korrel8r-rag

# Fetch latest korrel8r docs
./generate-docs.sh

# Build FAISS vector DB and push amd64 image
./build-and-push.sh quay.io/<your-repo>/korrel8r-rag:latest
```

> **Note:** The builder stage runs natively on ARM (Apple Silicon), the final image targets `linux/amd64` for OCP. See the [korrel8r-rag](https://github.com/JonkeyGuan/korrel8r-rag) repo for details.

### 4.4 Configure OLSConfig

```bash
KORREL8R_URL=$(oc get routes/korrel8r -n korrel8r -o template='https://{{.spec.host}}')

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
    rag:
    - image: quay.io/jonkey/korrel8r-rag:latest
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

> **rag**: The BYO Knowledge feature (Technology Preview) loads the korrel8r FAISS vector DB to provide context for LLM responses. Remove the `rag` section if you skipped step 4.3.

Wait for Lightspeed to be ready:

```bash
oc wait olsconfig cluster --for=jsonpath='{.status.overallStatus}'=Ready --timeout=300s -n openshift-lightspeed
```

### 4.5 Verify RAG is Loaded

```bash
oc logs deployment/lightspeed-app-server -n openshift-lightspeed -c lightspeed-service-api \
  | grep -i "vector index"
```

Expected:

```
Loading vector index #0 from quay.io/jonkey/korrel8r-rag:latest...
Vector index #0 is loaded.
All indexes are loaded.
```

## 5. Create Test Workloads (Optional)

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

## 6. Verification

### 6.1 Verify Korrel8r MCP Tools

After making a query in Lightspeed, check logs:

```bash
oc logs deployment/lightspeed-app-server -n openshift-lightspeed -c lightspeed-service-api \
  --tail=50 | grep "Loaded.*tools"
```

Expected:

```
Loaded 24 tools from MCP server 'openshift'
Loaded 8 tools from MCP server 'korrel8r'
```

### 6.2 Verify Signal Correlation Panel

1. Open the OpenShift console
2. Click **Signal Correlation** in the app launcher (grid icon, top-right)
3. The troubleshooting panel should open

### 6.3 Test with Lightspeed

Open the Lightspeed chat and try:

```
find error pod in demo-troubleshoot ns, and show correlated signals and show it in console
```

## Known Issues

| Issue | Root Cause | Workaround |
|-------|-----------|------------|
| MiniMax-M3 shows `<think>` tags | M3 is a reasoning model that outputs thinking traces | Cosmetic only; tool calling works correctly. Alternative: use MiniMax-Text-01 (no `<think>` but weaker tool calling) |
| `create_neighbors_graph` failures | Model generates incorrect korrel8r query syntax | Retry; RAG knowledge base improves accuracy |

## Cleanup

```bash
# Remove test workloads
oc delete namespace demo-troubleshoot

# Remove Lightspeed
oc delete olsconfig cluster -n openshift-lightspeed
oc delete subscription lightspeed-operator -n openshift-lightspeed
oc delete namespace openshift-lightspeed

# Remove troubleshooting panel
helm uninstall troubleshooting-panel-console-plugin -n troubleshooting-panel-console-plugin
oc delete namespace troubleshooting-panel-console-plugin

# Remove korrel8r route
oc delete route korrel8r -n korrel8r

# Remove console plugin registration
oc patch consoles.operator.openshift.io cluster --type=json \
  -p='[{"op":"test","path":"/spec/plugins","value":["troubleshooting-panel-console-plugin"]},{"op":"remove","path":"/spec/plugins"}]'
```
