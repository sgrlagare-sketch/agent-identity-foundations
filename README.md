# Agent Identity Foundations (Lab 2) — Step-by-Step Manual

This repository contains the refactored **GitHub Specialist Agent** (deployed to Vertex AI Agent Runtime with Agent Identity) and **Salesforce Specialist Agent** (deployed to Cloud Run), along with a complete step-by-step manual for moving external tool credentials out of agent `.env` files and into **Google Cloud Agent Identity Auth Manager** (`agentidentity.googleapis.com`).

---

## Architecture Overview

| Agent | Runtime Target | Principal Identity Type | Auth Provider Type | Credential Delivery Mechanism |
| :--- | :--- | :--- | :--- | :--- |
| **`github-agent`** | Vertex AI Agent Runtime (Reasoning Engine) | Federated SPIFFE Principal (`principal://agents.global.org-...`) | API Key (`github-mcp-auth-provider`) | `McpToolset` + `GcpAuthProviderScheme` + `header_provider` (`Authorization: Bearer <token>`) |
| **`salesforce-agent`** | Cloud Run | Cloud Run Service Account (`serviceAccount:<PROJECT_NUMBER>-compute@...`) | 2-Legged OAuth / Client Credentials (`salesforce-2lo-auth-provider`) | `AuthenticatedFunctionTool` + `AuthConfig` + injected `credential: AuthCredential` |

---

## Task 1: Prepare Your Environment for Agent Identity

### 1.1 Set Project and Location Variables

In Cloud Shell, configure your project, location, and region variables:

```bash
export PROJECT_ID=$(gcloud config get-value project)
export LOCATION=us-central1
export REGION=us-central1
gcloud config set project ${PROJECT_ID}
export PROJECT_NUMBER=$(gcloud projects describe ${PROJECT_ID} --format='value(projectNumber)')
export ORGANIZATION_ID=$(gcloud projects get-ancestors ${PROJECT_ID} --format=json | jq -r '.[] | select(.type=="organization") | .id')
echo "Project: ${PROJECT_ID} (${PROJECT_NUMBER}), Org: ${ORGANIZATION_ID}, Location: ${LOCATION}, Agents: ${REGION}"
```

### 1.2 Download Setup Scripts and Agent Code

```bash
gcloud storage cp -r gs://${PROJECT_ID}-bucket/apex_labs .
gcloud storage cp -r gs://${PROJECT_ID}-bucket/agents ~/agents
export PATH="$HOME/.local/bin:$PATH"
```

### 1.3 Configure `.env` for `github-agent`

Create `~/agents/github-agent/.env` without any hardcoded `GITHUB_PERSONAL_ACCESS_TOKEN`:

```bash
cd ~/agents/github-agent
cat << EOF > .env
GOOGLE_CLOUD_PROJECT=$PROJECT_ID
GOOGLE_CLOUD_LOCATION=global
GOOGLE_GENAI_USE_VERTEXAI=True
MODEL=gemini-3.5-flash
GITHUB_MCP_URL=https://api.githubcopilot.com/mcp/
GITHUB_AUTH_PROVIDER_URI=projects/$PROJECT_ID/locations/$LOCATION/authProviders/github-mcp-auth-provider
EOF
```

### 1.4 Enable Required Google Cloud APIs

Enable the APIs required for Agent Identity, Auth Manager, Vertex AI Agent Runtime, and Cloud Run:

```bash
gcloud services enable \
  iam.googleapis.com \
  agentidentity.googleapis.com \
  aiplatform.googleapis.com \
  agentidentitycredentials.googleapis.com \
  iamcredentials.googleapis.com \
  sts.googleapis.com \
  run.googleapis.com \
  cloudbuild.googleapis.com \
  artifactregistry.googleapis.com \
  logging.googleapis.com \
  cloudtrace.googleapis.com \
  --project=${PROJECT_ID}
```

Ensure the Vertex AI Reasoning Engine service agent exists and has baseline model/telemetry permissions:

```bash
gcloud beta services identity create --service=aiplatform.googleapis.com --project=${PROJECT_ID}

RE_SA="service-${PROJECT_NUMBER}@gcp-sa-aiplatform-re.iam.gserviceaccount.com"
for role in roles/aiplatform.user roles/logging.logWriter roles/cloudtrace.agent; do
  gcloud projects add-iam-policy-binding ${PROJECT_ID} \
    --member="serviceAccount:${RE_SA}" \
    --role="${role}" \
    --condition=None
done
```

---

## Task 2: Create the GitHub API-Key Auth Provider

### 2.1 Store Your GitHub Personal Access Token in a Shell Variable

```bash
read -rs GITHUB_PAT && echo
```

### 2.2 Create the API-Key Auth Provider

```bash
gcloud alpha agent-identity auth-providers create github-mcp-auth-provider \
  --project=${PROJECT_ID} \
  --location=${LOCATION} \
  --api-key="${GITHUB_PAT}"
```

### 2.3 Verify the Auth Provider

```bash
gcloud alpha agent-identity auth-providers list \
  --project=${PROJECT_ID} \
  --location=${LOCATION}

export GITHUB_AUTH_PROVIDER_URI="projects/${PROJECT_ID}/locations/${LOCATION}/authProviders/github-mcp-auth-provider"
echo ${GITHUB_AUTH_PROVIDER_URI}
```

---

## Task 3: Move the GitHub Agent onto the Auth Provider

### 3.1 Refactor `~/agents/github-agent/agent/agent.py`

Update [`github-agent/agent/agent.py`](github-agent/agent/agent.py) to:
1. Register `GcpAuthProvider()` with `CredentialManager`:
   ```python
   CredentialManager.register_auth_provider(GcpAuthProvider())
   ```
2. Define the `GcpAuthProviderScheme`:
   ```python
   github_auth_scheme = GcpAuthProviderScheme(name=GITHUB_AUTH_PROVIDER_URI)
   ```
3. Build and return the `McpToolset` wired with `auth_scheme`, `tool_filter`, and `header_provider`:
   ```python
   toolset = McpToolset(
       connection_params=StreamableHTTPConnectionParams(url=GITHUB_MCP_URL),
       auth_scheme=github_auth_scheme,
       tool_filter=GITHUB_TOOL_FILTER,
       header_provider=github_header_provider,
   )
   return toolset
   ```

### 3.2 Deploy `github-agent` with Agent Identity Enabled

Run `deploy_with_identity.sh` to provision the Reasoning Engine with `identityType: AGENT_IDENTITY` first and then deploy the source code in a separate update:

```bash
cd ~/agents/github-agent
chmod 777 ./deploy_with_identity.sh
REGION=us-central1 A2A_APP_NAME=agent AUTH_PROVIDER_NAME=github-mcp-auth-provider DISPLAY_NAME=github-agent-v2 ./deploy_with_identity.sh
```

Record the `Effective identity` and `Reasoning Engine ID` output by the script:

```bash
export GITHUB_ENGINE_ID="<REASONING_ENGINE_ID>"
export GITHUB_AGENT_IDENTITY="principal://agents.global.org-${ORGANIZATION_ID}.system.id.goog/resources/aiplatform/projects/${PROJECT_NUMBER}/locations/${REGION}/reasoningEngines/${GITHUB_ENGINE_ID}"
export GITHUB_AGENT_URL="https://${REGION}-aiplatform.googleapis.com/v1/projects/${PROJECT_NUMBER}/locations/${REGION}/reasoningEngines/${GITHUB_ENGINE_ID}"
```

### 3.3 Grant the Agent's SPIFFE Principal Access to the Auth Provider

```bash
gcloud alpha agent-identity auth-providers add-iam-policy-binding github-mcp-auth-provider \
  --project=${PROJECT_ID} \
  --location=${LOCATION} \
  --role="roles/agentidentity.user" \
  --member="${GITHUB_AGENT_IDENTITY}"

# Optional: grant your user account access for local development/testing
gcloud alpha agent-identity auth-providers add-iam-policy-binding github-mcp-auth-provider \
  --project=${PROJECT_ID} \
  --location=${LOCATION} \
  --role="roles/agentidentity.user" \
  --member="user:$(gcloud config get-value account)"
```

### 3.4 Test the GitHub Agent over A2A

```bash
agents-cli run --url ${GITHUB_AGENT_URL} --mode a2a --app-name agent "List all my open GitHub issues"
```

---

## Task 4: Create the Salesforce 2LO Auth Provider

### 4.1 Set Salesforce Credentials in Shell Variables

```bash
export SF_DOMAIN="YOUR_SALESFORCE_MY_DOMAIN"   # e.g. customization-power-8564.my.salesforce.com
export SF_CLIENT_ID="YOUR_CONSUMER_KEY"
read -rs SF_CLIENT_SECRET && echo
```

### 4.2 Create the 2-Legged OAuth (2LO) Auth Provider

```bash
gcloud alpha agent-identity auth-providers create salesforce-2lo-auth-provider \
  --project=${PROJECT_ID} \
  --location=${LOCATION} \
  --two-legged-oauth-client-id="${SF_CLIENT_ID}" \
  --two-legged-oauth-client-secret="${SF_CLIENT_SECRET}" \
  --two-legged-oauth-token-url="https://${SF_DOMAIN}/services/oauth2/token"
```

### 4.3 Verify the Auth Provider

```bash
gcloud alpha agent-identity auth-providers list \
  --project=${PROJECT_ID} \
  --location=${LOCATION}

export SALESFORCE_AUTH_PROVIDER_URI="projects/${PROJECT_ID}/locations/${LOCATION}/authProviders/salesforce-2lo-auth-provider"
echo ${SALESFORCE_AUTH_PROVIDER_URI}
```

---

## Task 5: Update the Salesforce Agent to Use the Auth Provider

### 5.1 Refactor `~/agents/salesforce-agent/app/tools.py`

Update [`salesforce-agent/app/tools.py`](salesforce-agent/app/tools.py) to:
1. Register `GcpAuthProvider()` and create `salesforce_auth_config`:
   ```python
   CredentialManager.register_auth_provider(GcpAuthProvider())
   salesforce_auth_config = AuthConfig(
       auth_scheme=GcpAuthProviderScheme(name=SALESFORCE_AUTH_PROVIDER_URI)
   )
   ```
2. Accept `credential: AuthCredential` in `search_salesforce` and extract the token:
   ```python
   def search_salesforce(query: str, credential: AuthCredential) -> dict:
       access_token = _extract_token(credential)
   ```
3. Return `AuthenticatedFunctionTool`:
   ```python
   return AuthenticatedFunctionTool(
       func=search_salesforce, auth_config=salesforce_auth_config
   )
   ```

### 5.2 Refactor `~/agents/salesforce-agent/app/agent.py` and `salesforce_client.py`

- In [`salesforce-agent/app/agent.py`](salesforce-agent/app/agent.py), initialize `SalesforceClient` with only `domain=SALESFORCE_DOMAIN` (no `client_id` or `client_secret`).
- In [`salesforce-agent/app/salesforce_client.py`](salesforce-agent/app/salesforce_client.py), remove `authenticate()` and derive `self.instance_url = f"https://{self.domain}"` from the My Domain host.

### 5.3 Update `.env` and Dependencies

Update `~/agents/salesforce-agent/.env` so it contains only non-secret configuration and `SALESFORCE_AUTH_PROVIDER_URI`:

```bash
cd ~/agents/salesforce-agent
cat << EOF > .env
GOOGLE_CLOUD_PROJECT=${PROJECT_ID}
GOOGLE_CLOUD_LOCATION=global
GOOGLE_GENAI_USE_VERTEXAI=True
MODEL=gemini-3.5-flash
SALESFORCE_DOMAIN=${SF_DOMAIN}
SALESFORCE_AUTH_PROVIDER_URI=projects/${PROJECT_ID}/locations/${LOCATION}/authProviders/salesforce-2lo-auth-provider
EOF
```

Add the `agent-identity` extra to `pyproject.toml` and update `uv.lock`:

```bash
cd ~/agents/salesforce-agent
sed -i 's/google-adk\[gcp\]/google-adk[gcp,agent-identity]/' pyproject.toml
uv lock
```

### 5.4 Deploy `salesforce-agent` to Cloud Run

```bash
cd ~/agents/salesforce-agent
agents-cli deploy --project ${PROJECT_ID} --region ${REGION} \
  --no-confirm-project \
  --max-instances 4 \
  --update-env-vars "$(grep -v '^#' .env | grep -v '^$' | grep -v '^GOOGLE_CLOUD_PROJECT' | paste -sd,)"
```

### 5.5 Grant the Cloud Run Service Account Access to the Auth Provider

```bash
export SF_SA=$(gcloud run services describe salesforce-agent --region ${REGION} \
  --format='value(spec.template.spec.serviceAccountName)')
echo ${SF_SA}

gcloud alpha agent-identity auth-providers add-iam-policy-binding salesforce-2lo-auth-provider \
  --project=${PROJECT_ID} \
  --location=${LOCATION} \
  --role="roles/agentidentity.user" \
  --member="serviceAccount:${SF_SA}"

gcloud alpha agent-identity auth-providers add-iam-policy-binding salesforce-2lo-auth-provider \
  --project=${PROJECT_ID} \
  --location=${LOCATION} \
  --role="roles/agentidentity.user" \
  --member="user:$(gcloud config get-value account)"
```

### 5.6 Disable IAP and Test the Salesforce Agent over A2A

```bash
gcloud run services update salesforce-agent --region=${REGION} --no-iap

export SF_URL=$(gcloud run services describe salesforce-agent --region ${REGION} --format='value(status.url)')
cd ~/agents/salesforce-agent
agents-cli run --url ${SF_URL} --mode a2a "Search for any architecture decision records about timeouts."
```
