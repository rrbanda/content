# MCP Hub-Spoke Architecture — Demo Narrative

> **Format:** Tell-Show-Tell. Each section opens with what you're about to show,
> runs the live command, then explains what just happened.

---

## Setup

Set these variables once at the start of the demo. Everything below uses them.

```bash
export MCP_GW=mcp-gateway-mcp-system.apps.rosa.afred-34-test.js4d.p3.openshiftapps.com
export KC_HOST=mcp-keycloak-keycloak.apps.rosa.afred-34-test.js4d.p3.openshiftapps.com
export AGENT_URL=https://rhoai-ocp-agent-ocp-agent-ocp-agent.apps.rosa.afred-34-test.js4d.p3.openshiftapps.com
export AUDIENCE="https://oidc.op1.openshiftapps.com/2juqbss0c7be4ihcm2cb9okmq34rav87"
```

---

## Part 1: MCP Gateway with ServiceAccount Token (On-Cluster Auth)

### TELL: What we're about to do

We're going to authenticate as an on-cluster agent using a Kubernetes ServiceAccount
token and walk through the full MCP flow: initialize a session, discover tools, and
call a tool that returns real cluster data. This is the path an agent running on the
same OpenShift cluster would take.

### SHOW: Create a ServiceAccount token

```bash
SA_TOKEN=$(oc create token mcp-viewer -n ocp-mcp-server \
  --audience="${AUDIENCE}")
echo "Token length: ${#SA_TOKEN} chars"
```

This creates a short-lived token for the `mcp-viewer` ServiceAccount in the
`ocp-mcp-server` namespace. The `--audience` flag is critical — it must match
the cluster's OIDC issuer URL, which is what Authorino checks during
`kubernetesTokenReview`.

### SHOW: Step 1 — Initialize (create a session)

```bash
curl -sk -D /tmp/headers.txt \
  -X POST "https://${MCP_GW}/mcp" \
  -H "Authorization: Bearer ${SA_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "initialize",
    "params": {
      "protocolVersion": "2025-03-26",
      "capabilities": {},
      "clientInfo": {"name": "demo-cli", "version": "1.0.0"}
    }
  }'
```

**What happens behind the scenes:**
1. Request hits the OpenShift Route → TLS terminates → Envoy Gateway on port 8080
2. MCPRouter (ext_proc) parses JSON-RPC, identifies "initialize" — no routing headers needed
3. Authorino WASM validates the SA token via kubernetesTokenReview against the K8s API
4. Limitador WASM checks rate limits (30/10s burst, 500/1h sustained)
5. Request reaches the MCPBroker — it creates a session, connects to registered MCP servers,
   and returns the aggregated tool catalog

### SHOW: Capture the session ID

```bash
SESSION=$(grep -i "mcp-session-id" /tmp/headers.txt | tr -d '\r' | awk '{print $2}')
echo "Session: ${SESSION:0:30}..."
```

Every subsequent request must include this session ID.

### SHOW: Step 2 — List available tools

```bash
curl -sk \
  -X POST "https://${MCP_GW}/mcp" \
  -H "Authorization: Bearer ${SA_TOKEN}" \
  -H "Content-Type: application/json" \
  -H "Mcp-Session-Id: ${SESSION}" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}' \
  | python3 -c "
import sys, json
d = json.load(sys.stdin)
tools = d['result']['tools']
print(f'Tools available: {len(tools)}')
for t in tools[:8]:
    print(f'  - {t[\"name\"]}')
print(f'  ... and {len(tools)-8} more')
"
```

**What happens:** Same path as initialize — goes through MCPRouter, auth, rate limit,
then to the MCPBroker which returns the aggregated catalog. Each tool is prefixed
with `openshift_` to identify which backend MCP server owns it.

### SHOW: Step 3 — Call a tool (list hub namespaces)

```bash
curl -sk \
  -X POST "https://${MCP_GW}/mcp" \
  -H "Authorization: Bearer ${SA_TOKEN}" \
  -H "Content-Type: application/json" \
  -H "Mcp-Session-Id: ${SESSION}" \
  -d '{
    "jsonrpc":"2.0",
    "id":3,
    "method":"tools/call",
    "params":{
      "name":"openshift_namespaces_list",
      "arguments":{}
    }
  }'
```

**What happens — and this is the key architectural difference:**
1. MCPRouter parses the request, sees "tools/call", reads the tool name
2. Router sets the `:authority` header to `openshift-mcp.mcp.local` (the target MCP server)
3. Router strips the `openshift_` prefix so the MCP server receives `namespaces_list`
4. Auth and rate limiting happen
5. Envoy routes **directly to the MCP Server** using the `:authority` header — the MCPBroker
   is NOT involved in tool calls
6. MCP Server calls the hub's K8s API and returns real namespace data

### SHOW: Step 4 — Call a tool on a spoke cluster

```bash
curl -sk \
  -X POST "https://${MCP_GW}/mcp" \
  -H "Authorization: Bearer ${SA_TOKEN}" \
  -H "Content-Type: application/json" \
  -H "Mcp-Session-Id: ${SESSION}" \
  -d '{
    "jsonrpc":"2.0",
    "id":4,
    "method":"tools/call",
    "params":{
      "name":"openshift_namespaces_list",
      "arguments":{"context":"sandbox602"}
    }
  }'
```

**What happens:** Same flow as step 3, but this time the `context: sandbox602` argument
tells the MCP Server to select the sandbox602 kubeconfig context from the mounted
`spoke-kubeconfig` Secret. The MCP Server uses the `mcp-hub-reader` SA token
(cluster-reader role on the spoke) to call the spoke's K8s API. The Gateway, Router,
and auth layer don't know or care that this is a different cluster — they're spoke-agnostic.

### SHOW: Step 5 — Verify unauthenticated is blocked

```bash
curl -sk -w "\nHTTP: %{http_code}\n" \
  -X POST "https://${MCP_GW}/mcp" \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc":"2.0",
    "id":1,
    "method":"initialize",
    "params":{
      "protocolVersion":"2025-03-26",
      "capabilities":{},
      "clientInfo":{"name":"anon","version":"1.0.0"}
    }
  }'
```

Expected: `HTTP: 401` with a JSON error message. The request never reaches the
MCPBroker or MCP Server — Authorino rejects it at the Gateway.

### TELL: What we just proved

We walked through the complete MCP flow using a ServiceAccount token: session
initialization through the Broker, tool discovery, direct tool execution on the
hub, cross-cluster tool execution on the spoke, and unauthenticated rejection.
This is the path any on-cluster agent takes.

---

## Part 2: MCP Gateway with Keycloak JWT (External Agent Auth)

### TELL: What we're about to do

Now we'll authenticate as an external agent — one that doesn't have access to
OpenShift ServiceAccount tokens because it's running on GKE, EKS, or another
platform. This agent uses Keycloak to get a JWT.

### SHOW: Get a JWT from Keycloak

First, we get an admin token to look up the client secret (in production this
comes from Vault, not from the admin API):

```bash
KC_ADMIN_PASS=$(oc get secret mcp-keycloak-initial-admin -n keycloak \
  -o jsonpath='{.data.password}' | base64 -d)

ADMIN_TOKEN=$(curl -sk -X POST "https://${KC_HOST}/realms/master/protocol/openid-connect/token" \
  -d "client_id=admin-cli" \
  -d "username=temp-admin" \
  -d "password=${KC_ADMIN_PASS}" \
  -d "grant_type=password" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

CLIENT_UUID=$(curl -sk -H "Authorization: Bearer ${ADMIN_TOKEN}" \
  "https://${KC_HOST}/admin/realms/openshift/clients" \
  | python3 -c "import sys,json; [print(c['id']) for c in json.load(sys.stdin) if c['clientId']=='adk-agent-client']")

CLIENT_SECRET=$(curl -sk -H "Authorization: Bearer ${ADMIN_TOKEN}" \
  "https://${KC_HOST}/admin/realms/openshift/clients/${CLIENT_UUID}/client-secret" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['value'])")
```

Now get the JWT the way a real external agent would — `client_credentials` grant:

```bash
KC_JWT=$(curl -sk -X POST \
  "https://${KC_HOST}/realms/openshift/protocol/openid-connect/token" \
  -d "grant_type=client_credentials" \
  -d "client_id=adk-agent-client" \
  -d "client_secret=${CLIENT_SECRET}" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

echo "JWT length: ${#KC_JWT} chars"
```

**What just happened:** The agent called Keycloak's token endpoint with
`client_credentials` — no user login, no browser, just machine-to-machine
authentication. Keycloak returned a JWT valid for 30 minutes.

### SHOW: Call the MCP Gateway with the Keycloak JWT

```bash
curl -sk -w "\nHTTP: %{http_code}\n" \
  -X POST "https://${MCP_GW}/mcp" \
  -H "Authorization: Bearer ${KC_JWT}" \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc":"2.0",
    "id":1,
    "method":"initialize",
    "params":{
      "protocolVersion":"2025-03-26",
      "capabilities":{},
      "clientInfo":{"name":"external-agent","version":"1.0.0"}
    }
  }'
```

Expected: `HTTP: 200` — same response as the SA token path. The Gateway accepted
the Keycloak JWT because the AuthPolicy has dual authentication: it accepts
**either** a valid SA token **or** a valid Keycloak JWT.

### TELL: What we just proved

The same MCP Gateway accepts both authentication methods. Keycloak interfaces
with the Gateway (Authorino validates the JWT against Keycloak's JWKS endpoint),
NOT with individual MCP servers. An external agent on any platform can get a JWT
from Keycloak and access the full MCP tool catalog — same tools, same rate limits,
same security.

---

## Part 3: The ADK Agent (AI-Powered Operations)

### TELL: What we're about to do

Now let's see what it looks like when an actual AI agent uses the MCP Gateway.
We have a Google ADK agent running on the hub cluster that connects to the
MCP Gateway using Keycloak JWT. It uses Gemini 3.6 Flash as the LLM.
When you ask it a question, the LLM decides which MCP tool to call, the tool
executes against the real K8s API, and the LLM summarizes the result.

### SHOW: Verify the agent is running

```bash
oc get pods -n ocp-agent -l app.kubernetes.io/name=ocp-agent
```

### SHOW: Check the agent's MCP auth config

```bash
oc set env deployment/rhoai-ocp-agent-ocp-agent -n ocp-agent --list \
  | grep -E "MCP_GATEWAY|KEYCLOAK|MCP_CLIENT_ID|MODEL_ID"
```

This shows:
- `MCP_GATEWAY_URL` — the MCP Gateway endpoint
- `KEYCLOAK_TOKEN_URL` — where the agent gets its JWT
- `MCP_CLIENT_ID` — `adk-agent-client` (Keycloak client)
- `MODEL_ID` — `gemini-3.6-flash`

### SHOW: Open the Agent Web UI

Open in browser:
```
https://rhoai-ocp-agent-ocp-agent-ocp-agent.apps.rosa.afred-34-test.js4d.p3.openshiftapps.com
```

### SHOW: Ask the agent about the hub cluster

Type in the chat:
```
What namespaces exist on this cluster?
```

**What happens behind the scenes:**
1. Your message goes to the Gemini 3.6 Flash LLM
2. LLM decides to call the `openshift_namespaces_list` tool
3. Agent's `mcp_auth.py` gets a Keycloak JWT (cached, refreshes every 30 min)
4. Agent sends `tools/call` to the MCP Gateway with `Authorization: Bearer <JWT>`
5. Gateway → MCPRouter → Authorino → Limitador → MCP Server → K8s API
6. Real namespace data returns → LLM summarizes it in plain English

### SHOW: Ask about a specific namespace

```
What pods are running in the ocp-mcp-server namespace?
```

The agent calls `openshift_pods_list` with `namespace: ocp-mcp-server` — you should
see the MCP server pod itself in the response.

### SHOW: Ask about the spoke cluster

```
List namespaces on the sandbox602 cluster
```

The agent calls `openshift_namespaces_list` with `context: sandbox602` — same tool,
different cluster. The MCP Server selects the spoke kubeconfig context and queries
the spoke's K8s API.

### SHOW: Ask about alerts

```
Are there any alerts firing on this cluster?
```

The agent calls `openshift_get_alerts` — Prometheus alerts from the hub cluster.

### TELL: What we just proved

An AI agent, running on OpenShift, authenticated via Keycloak JWT, called real
MCP tools through the Gateway, and returned actual cluster data from both the
hub and a spoke cluster. The agent didn't use kubectl, didn't have overprivileged
access, and every call went through authentication, rate limiting, and audit.
This is what safe, controlled AI agent access to a cluster fleet looks like.

---

## Part 4: Security Verification

### TELL: What we're about to show

Let's verify the security layers are actually working — not just trust that they are.

### SHOW: Unauthenticated request → 401

```bash
curl -sk -w "\nHTTP: %{http_code}\n" \
  -X POST "https://${MCP_GW}/mcp" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-03-26","capabilities":{},"clientInfo":{"name":"anon","version":"1.0.0"}}}'
```

Expected: `401 Unauthorized`. Authorino rejected it at the Gateway.

### SHOW: AuthPolicy is enforced

```bash
oc get authpolicy mcp-auth-policy -n mcp-system \
  -o jsonpath='{.status.conditions}' \
  | python3 -c "
import sys, json
for c in json.load(sys.stdin):
    print(f'  {c[\"type\"]}: {c[\"status\"]}')
"
```

Expected: `Accepted: True`, `Enforced: True`.

### SHOW: Rate limits are configured

```bash
oc get ratelimitpolicy mcp-rate-limit -n mcp-system \
  -o jsonpath='{.spec.defaults.limits}' \
  | python3 -c "
import sys, json
for name, cfg in json.load(sys.stdin).items():
    rates = cfg.get('rates', [])
    for r in rates:
        print(f'  {name}: {r[\"limit\"]} per {r[\"window\"]}')
"
```

### SHOW: MCP Server is read-only

```bash
oc get configmap -n ocp-mcp-server -o yaml \
  | grep -A2 "read_only\|denied_resources" | head -10
```

### TELL: What we just proved

Authentication is enforced (401 on no token), the AuthPolicy is active and enforced,
rate limits are configured at three tiers, and the MCP server is read-only with
denied resources. Every layer is working.

---

## Demo Summary

| What | How | Result |
|------|-----|--------|
| SA token → MCP Gateway | `oc create token` + `curl` | 200 — session, tools, real data |
| Keycloak JWT → MCP Gateway | `client_credentials` + `curl` | 200 — same tools, same data |
| No token → MCP Gateway | `curl` without Authorization | 401 — blocked at Gateway |
| Hub namespace list | `tools/call` without context | Real hub data |
| Spoke namespace list | `tools/call` with `context: sandbox602` | Real spoke data |
| AI Agent → MCP Gateway | ADK web UI, natural language | LLM calls tools, returns answers |
| Security verification | AuthPolicy, RateLimit, read-only | All enforced |
