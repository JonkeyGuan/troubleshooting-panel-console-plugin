# Troubleshooting Panel + Lightspeed + Korrel8r Setup Guide

> Tested on OCP 4.22.6 with Lightspeed Operator v1.1.1 and Korrel8r v0.11.6

## Prerequisites

- OpenShift cluster 4.22+
- `oc` and `helm` CLIs installed
- Logged in as cluster-admin (`oc login`)
- Podman or Docker available locally (only if building custom images)
- LLM backend with Function Calling support (MiniMax or LiteLLM-compatible)

Clone the repo (needed for Helm chart and optional image build):

```bash
git clone https://github.com/openshift/troubleshooting-panel-console-plugin.git
cd troubleshooting-panel-console-plugin
```

## 1. Korrel8r

### 1.1 Install Korrel8r Operator

```bash
oc apply -f - <<'EOF'
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: korrel8r
  namespace: openshift-operators
spec:
  channel: alpha
  name: korrel8r
  source: community-operators
  sourceNamespace: openshift-marketplace
  installPlanApproval: Automatic
EOF
```

Wait for the operator to be ready:

```bash
oc wait csv -n openshift-operators -l operators.coreos.com/korrel8r.openshift-operators --for=jsonpath='{.status.phase}'=Succeeded --timeout=120s
```

### 1.2 Fix kube-rbac-proxy Image (if needed)

The operator's default `gcr.io/kubebuilder/kube-rbac-proxy:v0.13.1` image may no longer be available. Fix:

```bash
oc set image deployment/korrel8r-controller-manager \
  kube-rbac-proxy=registry.k8s.io/kubebuilder/kube-rbac-proxy:v0.13.1 \
  -n openshift-operators
```

### 1.3 Create Korrel8r Instance

```bash
oc create namespace korrel8r

oc apply -f - <<'EOF'
apiVersion: korrel8r.openshift.io/v1alpha1
kind: Korrel8r
metadata:
  name: korrel8r
  namespace: korrel8r
spec: {}
EOF
```

### 1.4 Upgrade Korrel8r to v0.11.6

The operator may deploy an older version (v0.7.2). Upgrade to v0.11.6 for MCP and `/graphs/neighbors` support:

```bash
# Scale down the operator to prevent it from reverting
oc scale deployment korrel8r-controller-manager -n openshift-operators --replicas=0

# Update the image
oc set image deployment/korrel8r -n korrel8r \
  korrel8r=quay.io/korrel8r/korrel8r:v0.11.6
```

### 1.5 Grant tokenreviews Permission

Korrel8r v0.11.6 requires `tokenreviews` access:

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
  --serviceaccount=korrel8r:default
```

### 1.6 Create External Route

```bash
oc create route reencrypt --service=korrel8r -n korrel8r
```

Verify:

```bash
KORREL8R_URL=$(oc get routes/korrel8r -n korrel8r -o template='https://{{.spec.host}}')
TOKEN=$(oc whoami -t)
curl -sk "$KORREL8R_URL/api/v1alpha1/domains" -H "Authorization: Bearer $TOKEN" | python3 -m json.tool
```

## 2. Troubleshooting Panel Plugin

### 2.1 Build and Push Container Image (Optional)

Skip this step if using the pre-built image `quay.io/jonkey/troubleshooting-panel-console-plugin:latest` (public repo).

```bash
# Login to registries
podman login registry.redhat.io
podman login quay.io

# Build frontend, run tests, and build container image
REGISTRY_ORG=jonkey TAG=latest PUSH=1 make build-image
```

### 2.2 Deploy with Helm

```bash
IMAGE=quay.io/jonkey/troubleshooting-panel-console-plugin:latest
KORREL8R_NAMESPACE=korrel8r

helm upgrade --install troubleshooting-panel-console-plugin charts/openshift-console-plugin \
  --namespace troubleshooting-panel-console-plugin --create-namespace \
  --set plugin.image=$IMAGE \
  --set plugin.proxy[0].alias=korrel8r \
  --set plugin.proxy[0].endpoint.service.name=korrel8r \
  --set plugin.proxy[0].endpoint.service.namespace=$KORREL8R_NAMESPACE \
  --set plugin.proxy[0].endpoint.service.port=8443
```

### 2.3 Enable the Console Plugin

```bash
oc patch consoles.operator.openshift.io cluster --type=json \
  -p='[{"op":"add","path":"/spec/plugins/-","value":"troubleshooting-panel-console-plugin"}]'
```

### 2.4 Enable Agent Navigation Feature

```bash
oc set env deployment/troubleshooting-panel-console-plugin \
  TROUBLESHOOTING_PANEL_CONSOLE_PLUGIN_FEATURES=agent-navigation \
  -n troubleshooting-panel-console-plugin
```

> **Note:** The feature flag name must be exactly `agent-navigation` (with hyphen), not `enableagentnavigation`.

## 3. OpenShift Lightspeed

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
oc wait csv -n openshift-lightspeed -l operators.coreos.com/lightspeed-operator.openshift-lightspeed --for=jsonpath='{.status.phase}'=Succeeded --timeout=300s
```

### 3.2 Create LLM API Key Secret

```bash
# Option A: MiniMax (recommended — better tool-calling quality)
oc create secret generic minimax-api-key \
  --from-literal=apitoken=<YOUR_MINIMAX_API_KEY> \
  -n openshift-lightspeed

# Option B: LiteLLM proxy (e.g. gpt-oss-120b)
oc create secret generic litellm-api-key \
  --from-literal=apitoken=<YOUR_API_KEY> \
  -n openshift-lightspeed
```

### 3.3 Configure OLSConfig

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
    # Option A: MiniMax
    - name: minimax
      type: openai
      url: https://api.minimax.io/v1
      credentialsSecretRef:
        name: minimax-api-key
      models:
      - name: MiniMax-M3
    # Option B: LiteLLM proxy
    # - name: litellm
    #   type: openai
    #   url: https://<LITELLM_ENDPOINT>/v1/
    #   credentialsSecretRef:
    #     name: litellm-api-key
    #   models:
    #   - name: <MODEL_NAME>
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

> **maxIterations**: Default is 5, which is too low for complex prompts that require multiple tool calls (e.g. find pods → correlate → show in console). Set to 10 to allow the full tool chain to complete.

> **Important: MCP Auth Type Selection**
>
> | Type | Behavior | Use Case |
> |------|----------|----------|
> | `kubernetes` | Uses the requesting user's bearer token | **Recommended.** Supports `show_in_console` (same user identity as browser SSE session) |
> | `secret` | Uses a static token from a Secret | Tools load, but `show_in_console` fails due to session isolation |
> | `client` | Expects frontend to forward token | Broken in OLS v1.1.1 — console plugin doesn't send MCP client headers |

### 3.4 Wait for Lightspeed to be Ready

```bash
oc wait olsconfig cluster --for=jsonpath='{.status.overallStatus}'=Ready --timeout=300s -n openshift-lightspeed
```

## 4. Verification

### 4.1 Verify Korrel8r MCP Tools Load

After making a query in Lightspeed, check the logs:

```bash
oc logs deployment/lightspeed-app-server -n openshift-lightspeed --tail=50 | grep "Loaded.*tools"
```

Expected output:

```
Loaded 24 tools from MCP server 'openshift'
Loaded 8 tools from MCP server 'korrel8r'
```

The 8 Korrel8r MCP tools are:

| Tool | Description |
|------|-------------|
| `get_console` | Find out what data the user is looking at in the console |
| `show_in_console` | Display results in the console as a korrel8r query |
| `list_domains` | Discover available signal/resource domains |
| `list_domain_classes` | List classes (signal types) in a domain |
| `help` | Get examples of classes and query syntax for a domain |
| `create_goals_graph` | Search for correlations to a specific signal type (goal-directed) |
| `create_neighbors_graph` | Open-ended exploration of related objects (neighbourhood search) |
| `get_objects` | Get objects matching a korrel8r query |

### 4.2 Verify Agent Navigation

1. Open the OpenShift console
2. Click **Signal Correlation** in the app launcher (grid icon, top-right)
3. Click the **AI icon** in the troubleshooting panel toolbar
4. Toggle **Agent Navigation ON**
5. Verify status shows **Connected** (green)

### 4.3 Test with Lightspeed

Open the Lightspeed chat and try:

```
find error pod in demo-troubleshoot ns, and show correlated signals and show it in console
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

## Known Issues

| Issue | Root Cause | Workaround |
|-------|-----------|------------|
| `show_in_console` returns 400 Bad Request | LLM generates invalid Korrel8r query syntax | Use a more capable model (GPT-4o, Claude) or use an external agent (Claude Code) as MCP client |
| `client` auth type doesn't load Korrel8r tools | OLS v1.1.1 console plugin doesn't forward MCP client headers | Use `kubernetes` auth type instead |
| `get_console` returns "no console is connected" | Agent Navigation not enabled or SSE disconnected | Toggle Agent Navigation ON in the troubleshooting panel |
| Empty LLM responses (stuck on "...") | Model returns empty answer after tool calls | Try a different prompt or model with better tool-use support |

## Cleanup

```bash
# Remove test workloads
oc delete namespace demo-troubleshoot

# Remove troubleshooting panel
helm uninstall troubleshooting-panel-console-plugin -n troubleshooting-panel-console-plugin
oc delete namespace troubleshooting-panel-console-plugin

# Remove Lightspeed
oc delete olsconfig cluster -n openshift-lightspeed
oc delete subscription lightspeed-operator -n openshift-lightspeed
oc delete namespace openshift-lightspeed

# Remove Korrel8r
oc delete korrel8r korrel8r -n korrel8r
oc delete namespace korrel8r
oc scale deployment korrel8r-controller-manager -n openshift-operators --replicas=1
```
