# AI Scan Repos: Installation, Deployment, and Testing

This is the practical entry point for bringing up AI Scan Repos on Kubernetes and checking the MCP review engine. It follows the project's current K3s deployment. For a production deployment, use a reviewed, secured environment and the [deployment guide](docs/10-deployment-guide.md).

## 1. Know which repository does what

`ai-scan-repo` is the architecture and documentation repository. The application source and Kubernetes manifests are in the adjacent repositories:

| Repository | Contents |
|---|---|
| `ai-scanner-backend` | `review-api`, the shared MCP review pipeline, scanner code, and the `scanner-worker` Job image |
| `ai-scanner-gateway` | Login, session handling, settings, and gateway API |
| `ai-scanner-frontend` | Browser UI |
| `ai-scanner-backend-deployment` | Namespace, storage, PostgreSQL, review API, ingress, and K3s deployment guide |
| `ai-scanner-gateway-deployment` | Gateway Kubernetes manifests |
| `ai-scanner-frontend-deployment` | Frontend Kubernetes manifests |

The MCP component in `ai-scanner-backend/mcp-server` uses **stdio**. An MCP host such as LM Studio starts it as a subprocess/container; it is not an HTTP server or an independently addressable Kubernetes Service. In the Kubernetes platform, the `review-api` image embeds the shared MCP pipeline, and the API starts `scanner-worker` Jobs for reviews. Deploy the platform manifests for the in-cluster workflow; do not deploy `code-review-mcp` as a normal network Service.

This document is the end-to-end installation sequence. The other documents, such as the [K3s deployment runbook](../ai-scanner-backend-deployment/DEPLOY-K3S.md) and [MCP server guide](docs/15-mcp-server-documentation.md), remain available for troubleshooting and component-level details, but you can complete the normal installation by following the sections below in order.

## 2. Prerequisites

### Deployment at a glance

Follow these steps in order:

| Step | Go to |
|---|---|
| Prepare the server | [Section 3](#3-prepare-the-linuxk3s-server) |
| Clone the repositories | [Section 4](#4-check-out-the-application-and-manifest-repositories) |
| Configure model access and GitHub login | [Section 5](#5-configure-model-access-and-github-login) |
| Build images and set tags | [Sections 6-7](#6-build-the-container-images) |
| Create Kubernetes Secrets | [Section 8](#8-create-the-namespace-and-secrets) |
| Deploy and verify | [Sections 9-10](#9-render-and-deploy-the-kubernetes-manifests) |
| Test MCP | [Section 11](#11-test-the-mcp-component) |

### What you need

You need a Linux server, access to the six repositories, and admin access to a K3s cluster. Section 3 installs the tools on a new server. For reviews, choose either LM Studio (requires Tailscale) or a cloud model provider.  GitHub OAuth is required for web-app login.

### Install prerequisite packages

Run one command for your Linux distribution. **MCP tests require Python 3.10+.**

#### Ubuntu/Debian:

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl git openssl tar gzip python3 python3-pip python3-venv
```

#### RHEL/Rocky/AlmaLinux:

```bash
sudo yum install -y ca-certificates curl git openssl tar gzip python3 python3-pip
```

On older RHEL-family releases, install Python 3.10+ from an approved repository. K3s, Docker/Buildx, Helm, and Tailscale are installed separately in sections 3 and 5.

| Requirement | Check |
|---|---|
| K3s and kubectl | `kubectl get nodes` shows the target node as `Ready`. |
| Docker with Buildx | `docker buildx version` works; you can run Docker and `sudo k3s ctr`. |
| Git, OpenSSL, Python 3.10+, Helm 3 | `git --version`, `openssl version`, `python3 --version`, `helm version`. |
| Repository access | Can clone all six repositories; never put credentials in a URL or committed file. |
| GitHub OAuth | Create an OAuth App; its callback must match the URL configured in section 5. |
| Model provider | LM Studio over Tailscale, or a cloud provider configured after deployment. |

For an existing server, check the cluster and Python version:

```bash
kubectl get nodes
python3 --version
```

The node should be `Ready`. For a new server, install K3s and the tools in section 3 first.

## 3. Prepare the Linux/K3s server

Run these commands on the Linux server. If K3s or a tool is already installed, check it and skip its installation step. The example uses Ubuntu/Debian package commands.

### 3.1 Install K3s if this is a new server

Install K3s using the official installer, then configure `kubectl` for your login user:

```bash
curl -sfL https://get.k3s.io | sh -
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown "$USER":"$USER" ~/.kube/config
chmod 600 ~/.kube/config
kubectl get nodes
```

Continue when the node is `Ready`. The kubeconfig grants cluster-admin access: keep it private, do not commit it, and do not send it to anyone. On a remote admin workstation, securely copy the kubeconfig and replace its server address with a reachable K3s server address.

### 3.2 Install the image build tools

Docker and K3s use separate image stores. Docker builds the containers; later steps import them into K3s. If Docker is missing on this Ubuntu/RHEL server:

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker "$USER"
```

Log out and back in for the Docker group membership to take effect. **Verify Docker and Buildx**:

```bash
docker run --rm hello-world
docker buildx version
```

Install Helm 3 using the official installer if it is missing, then check `helm version`:

```bash
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

Finally verify cluster access and image import permissions:

```bash
kubectl config current-context
kubectl get nodes
sudo k3s ctr version
```

Only run installer scripts from sources you trust, on a server you administer. For a production cluster, follow your organization’s hardened installation process instead of these development-server examples.

## 4. Check out the application and manifest repositories

On the K3s server, clone the six repositories side by side under `~/app/scan-repo/`:

```bash
mkdir -p ~/app/scan-repo
cd ~/app/scan-repo
for repo in ai-scanner-backend ai-scanner-gateway ai-scanner-frontend \
            ai-scanner-backend-deployment ai-scanner-gateway-deployment \
            ai-scanner-frontend-deployment; do
  if [ ! -d "$repo" ]; then
    git clone "https://github.com/DevOpsCourse22/$repo.git"
  fi
done
```

If the repositories are private, authenticate through Git's prompt or your approved credential manager. Do not put a personal access token (PAT) in a clone URL or a committed file.

## 5. Configure model access and GitHub login

The default K3s configuration uses LM Studio on a machine in the same Tailscale network. The cluster reaches it through the Tailscale operator and the Kubernetes Service `lmstudio`. If the app will use a cloud provider instead, skip the LM Studio/Tailscale subsection and configure the provider through the admin Settings page after the application is deployed.

### 5.1 Start LM Studio and record its tailnet address

On the computer that runs LM Studio:

1. Install Tailscale on the computer running LM Studio and connect it to the same tailnet as the K3s server. Keep that computer awake, Tailscale connected, and the LM Studio server running while reviews are in progress. Otherwise, the cluster may lose access to the model and reviews may fail.
2. In LM Studio, load the model you intend to use, open Developer, start the server, and enable **Serve on Local Network**. The server must listen on port `1234`, not only on `127.0.0.1`.
3. Record the model computer's Tailscale IPv4 address:

   ```bash
   tailscale ip -4
   ```

4. In `ai-scanner-backend-deployment/review-api/lmstudio-egress.yaml`, Update the `tailscale.com/tailnet-ip` annotation to your address show above. The checked-in value is specific to the original deployment and will not necessarily match your machine.

From `~/app/scan-repo`, replace the example IP with the actual Tailscale IP of the computer running LM Studio:

```bash
sed -i 's/100\.93\.244\.39/<New-IP>/' ai-scanner-backend-deployment/review-api/lmstudio-egress.yaml
```

Replace **New-IP** with your Tailscale IP. Verify the change with:

```bash
grep 'tailscale.com/tailnet-ip' ai-scanner-backend-deployment/review-api/lmstudio-egress.yaml
```

### 5.2 Permit the private model connection

In the Tailscale admin console:

1. Make sure the K3s server is also signed in to the same tailnet. If Tailscale is not installed on this Ubuntu/Debian server, install and connect it:

  ```bash
  curl -fsSL https://tailscale.com/install.sh | sh
  sudo tailscale up
  tailscale ip -4
  ```

  Complete any sign-in URL that `tailscale up` prints. The server and model host should appear in `tailscale status`.  
  2. Open **Access controls** and review `ai-scanner-backend-deployment/tailscale-operator/tailnet-policy.hujson`. Adapt the policy to your tailnet; do not overwrite an existing policy blindly. It must allow the operator to own the `tag:k8s` proxy tag and allow that tag to reach only the LM Studio machine on TCP `1234`.  
  3. Under **Settings → Trust credentials**, create an OAuth client for the operator. Grant the permissions required by the checked-in `tailscale-operator/values.yaml` (General/Services, Devices/Core, and Keys/Auth Keys, with the `tag:k8s-operator` tag). Keep the client ID and secret for the next step.  
  4. On the K3s server, create the operator Secret and install the pinned Helm chart:

   ```bash
   kubectl create namespace tailscale
   read -rp "Tailscale OAuth client ID: " TS_ID
   read -rsp "Tailscale OAuth client secret: " TS_SECRET; echo
   kubectl -n tailscale create secret generic operator-oauth \
     --from-literal=client_id="$TS_ID" \
     --from-literal=client_secret="$TS_SECRET"
   unset TS_ID TS_SECRET

   helm repo add tailscale https://pkgs.tailscale.com/helmcharts
   helm repo update
   helm upgrade --install tailscale-operator tailscale/tailscale-operator \
     -n tailscale --version 1.102.4 \
     -f ~/app/scan-repo/ai-scanner-backend-deployment/tailscale-operator/values.yaml \
     --wait
   kubectl -n tailscale get pods
   ```

   The operator pod should be `Running`. The Secret is created outside Git and the credential is not printed. If the namespace or Secret already exists, inspect it instead of trying to create a duplicate.

### 5.3 Configure the application URL and GitHub OAuth

On the K3s server, get its Tailscale address (use the address through which your browser will reach the app):

```bash
tailscale ip -4
```

In `ai-scanner-gateway-deployment/deployment.yaml`, set `GATEWAY_FRONTEND_URL`, `GATEWAY_CORS_ORIGINS`, and `GITHUB_OAUTH_REDIRECT_URI` to the same address. 
`100.64.1.2` is an example of the K3s server’s Tailscale IP. Replace it with your server’s actual Tailscale IP.

In `deployment.yaml`, set:

- `GATEWAY_FRONTEND_URL` to `http://<your-server-ip>`
- `GATEWAY_CORS_ORIGINS` to the same URL
- `GITHUB_OAUTH_REDIRECT_URI` to that URL plus `/auth/github/callback`

For example, if the server’s IP is `100.80.5.10`, the callback is `http://100.80.5.10/auth/github/callback`. Enter that exact callback URL in the GitHub OAuth App settings too.

Create an OAuth App in GitHub under **Settings → Developer settings → OAuth Apps**:

| GitHub field | Value |
|---|---|
| Homepage URL | `http://<K3S_SERVER_TAILNET_IP>` |
| Authorization callback URL | `http://<K3S_SERVER_TAILNET_IP>/auth/github/callback` |

The callback URL must exactly match `GITHUB_OAUTH_REDIRECT_URI`, including scheme, path, and trailing-slash behavior. Save the client ID and secret for section 8. This sample deploys HTTP for a private tailnet demo; for internet-facing or production use, configure TLS and secure cookies before exposing the application.

For a cloud provider, do not put its API key in these manifests or in Git. After deployment, an authorized admin configures the provider through the app's Settings; `settings-key` created in [section 8](#8-create-the-namespace-and-secrets) encrypts the saved key.

## 6. Build the container images

Build on the K3s server. K3s/containerd has its own image store, so the images must be imported after Docker builds them. Use immutable source tags for the API, gateway, and frontend; the worker tag is referenced separately by the API manifest.

```bash
cd ~/app/scan-repo
API_TAG=$(git -C ai-scanner-backend rev-parse --short HEAD)
GW_TAG=$(git -C ai-scanner-gateway rev-parse --short HEAD)
FE_TAG=$(git -C ai-scanner-frontend rev-parse --short HEAD)

bash ai-scanner-backend/build.sh --tag "$API_TAG"
docker build -t "review-gateway:$GW_TAG" ai-scanner-gateway
docker build -t "review-frontend:$FE_TAG" ai-scanner-frontend
bash ai-scanner-backend/scanner-worker/build.sh --tag 0.5.0
```

Import the four images into K3s:

```bash
docker save "review-api:$API_TAG" "review-gateway:$GW_TAG" \
  "review-frontend:$FE_TAG" scanner-worker:0.5.0 \
  | sudo k3s ctr images import -
sudo k3s ctr images ls -q | grep -E 'review-|scanner-worker'
```

Confirm the exact tags appear in the image list. The worker tag must match `SCANNER_WORKER_IMAGE` in `ai-scanner-backend-deployment/review-api/deployment.yaml`; if you build a different worker tag, update that value and the test Job manifests too.

## 7. Set the deployment image tags

The three deployment repositories select their application image tags in their `kustomization.yaml` files. On the Linux K3s server:

```bash
sed -i "s/^    newTag: [^ ]*/    newTag: $API_TAG/" ai-scanner-backend-deployment/kustomization.yaml
sed -i "s/^    newTag: [^ ]*/    newTag: $GW_TAG/" ai-scanner-gateway-deployment/kustomization.yaml
sed -i "s/^    newTag: [^ ]*/    newTag: $FE_TAG/" ai-scanner-frontend-deployment/kustomization.yaml
grep -H newTag ai-scanner-*-deployment/kustomization.yaml
```

Each `newTag` must match the image built and imported above. For a GitOps/Argo CD deployment, commit and push these tag changes instead of editing the server's checkout; do not make competing manual changes to Argo-managed resources.

## 8. Create the namespace and secrets

Apply the namespace first. Secrets are created directly in Kubernetes and must not be committed to a repository:

```bash
cd ~/app/scan-repo/ai-scanner-backend-deployment
kubectl apply -f namespace.yaml
```

Create the database credential and gateway Secret once. This example keeps generated values out of the terminal output and uses a URL-safe database password:

```bash
PG_PW=$(openssl rand -hex 24)
kubectl -n ai-scan-repos create secret generic postgres-credentials \
  --from-literal=POSTGRES_USER=ai_scan_repos \
  --from-literal=POSTGRES_PASSWORD="$PG_PW" \
  --from-literal=POSTGRES_DB=ai_scan_repos

read -rsp "GitHub OAuth client ID: " GH_ID; echo
read -rsp "GitHub OAuth client secret: " GH_SECRET; echo
kubectl -n ai-scan-repos create secret generic gateway-secrets \
  --from-literal=database-url="postgresql+psycopg://ai_scan_repos:${PG_PW}@postgres:5432/ai_scan_repos" \
  --from-literal=github-oauth-client-id="$GH_ID" \
  --from-literal=github-oauth-client-secret="$GH_SECRET" \
  --from-literal=github-webhook-secret="$(openssl rand -hex 32)" \
  --from-literal=settings-key="$(openssl rand -hex 32)"
unset PG_PW GH_ID GH_SECRET
```

These `create` commands are for first-time setup. If a Secret already exists, do not recreate it; check its name and key sizes without printing values. Secret rotation should be performed through your approved secret-management process; the [K3s deployment runbook](../ai-scanner-backend-deployment/DEPLOY-K3S.md) has additional recovery notes.

```bash
kubectl -n ai-scan-repos describe secret postgres-credentials gateway-secrets
```

## 9. Render and deploy the Kubernetes manifests

Preview each Kustomize build before applying it. Apply the backend first because it creates the namespace; then deploy the gateway and UI:

Kustomization files locate at:
```bash
aviela@MacBook-M1-Syverse final-project % find . -type f -name "kustomization.yaml"
./ai-scanner-frontend-deployment/kustomization.yaml
./ai-scanner-backend-deployment/kustomization.yaml
./ai-scanner-gateway-deployment/kustomization.yaml
```
----

```bash
cd ~/app/scan-repo
kubectl kustomize ai-scanner-backend-deployment >/dev/null
kubectl kustomize ai-scanner-gateway-deployment >/dev/null
kubectl kustomize ai-scanner-frontend-deployment >/dev/null

kubectl apply -k ai-scanner-backend-deployment
kubectl apply -k ai-scanner-gateway-deployment
kubectl apply -k ai-scanner-frontend-deployment
```

Wait for workloads to become ready:

```bash
kubectl -n ai-scan-repos get pods -w
```

Expected workloads include the frontend, gateway, PostgreSQL, and `review-api`, each ready. Stop watching with Ctrl+C after they settle. If any pod is not ready, inspect its events and logs before continuing:

```bash
kubectl -n ai-scan-repos get pods
kubectl -n ai-scan-repos describe pod <pod-name>
kubectl -n ai-scan-repos logs <pod-name> --tail=100
```

## 10. Verify the deployed services

Replace `<K3S_SERVER_TAILNET_IP>` with the server address configured in the gateway and ingress manifests. Run these checks from the server:

```bash
curl -fsS http://<K3S_SERVER_TAILNET_IP>/health
curl -fsS http://<K3S_SERVER_TAILNET_IP>/ | head -3
kubectl auth can-i create jobs -n ai-scan-repos --as=system:serviceaccount:ai-scan-repos:review-api
kubectl auth can-i list pods -n ai-scan-repos --as=system:serviceaccount:ai-scan-repos:review-api
```

Expected: `/health` reports healthy, `/` returns the UI HTML, and the authorization checks print `yes` then `no` respectively. The API needs to create Jobs but should not have broad permission to list Pods.

When using LM Studio, verify its Tailscale-managed route and query the model list from inside the API pod:

```bash
kubectl -n ai-scan-repos get svc lmstudio
kubectl -n ai-scan-repos exec deploy/review-api -- python -c \
  "import json,urllib.request as u; print([m['id'] for m in json.load(u.urlopen('http://lmstudio:1234/v1/models',timeout=10))['data']])"
```

The Service should resolve to the Tailscale operator proxy and the query should print model IDs. If using a cloud provider, skip this LM Studio-specific check; verify the provider in the application's Settings and run the review smoke test below.

Then, from a device on the configured tailnet:

1. Open `http://<K3S_SERVER_TAILNET_IP>/` and sign in with GitHub.
2. Start a **quick** review of a small public repository (or an authorized private repository).
3. On the K3s server, watch for the worker Job:

   ```bash
   kubectl -n ai-scan-repos get jobs -w
   ```

4. Confirm the Job completes, the review reaches `completed`, and the UI displays the report with scanner findings. This verifies the browser, ingress, frontend, gateway, database, review API, worker Job, repository clone, and model connection together.

For lower-level API checks, port-forward the internal service in another terminal and query its liveness/readiness and model probe:

```bash
kubectl -n ai-scan-repos port-forward svc/review-api 8000:8000
```

```bash
curl -fsS http://127.0.0.1:8000/healthz
curl -fsS http://127.0.0.1:8000/readyz
curl -fsS 'http://127.0.0.1:8000/api/v1/health?probe=1'
```

`/healthz` and `/readyz` should return success. The model probe should report that the configured provider is reachable. Do not use the model probe for Kubernetes liveness/readiness probes; it depends on the external model and can trigger unnecessary restarts.

## 11. Test the MCP component

### Automated tests

From the `ai-scanner-backend` repository, install the MCP package's development dependencies and run its tests. These tests exercise the pipeline without requiring a live LLM:

```bash
cd ~/app/scan-repo/ai-scanner-backend
python3 -m venv .venv
. .venv/bin/activate
python -m pip install -e './mcp-server[dev]'
python -m pytest mcp-server/tests -q
```

The same source is embedded in `review-api` and used by the worker build. Rebuild/import the affected images and redeploy after changing it; rerun the end-to-end quick review from the previous section.

### MCP host smoke test

To exercise the actual MCP protocol, build the standalone image from the backend repository:

```bash
cd ~/app/scan-repo/ai-scanner-backend
bash mcp-server/build.sh --tag test
mkdir -p reports
```

Register the container command as an MCP server in your MCP host. The server speaks MCP over **stdio**, so the host must launch it with interactive stdin (`docker run --rm -i`); it does not listen on a TCP port. A Docker-based server entry has this shape (replace the host reports path with its absolute path):

```json
{
  "mcpServers": {
    "code-review": {
      "command": "docker",
      "args": [
        "run", "--rm", "-i",
        "--add-host=host.docker.internal:host-gateway",
        "-v", "/absolute/path/to/ai-scanner-backend/reports:/reports",
        "-e", "LMSTUDIO_BASE_URL=http://host.docker.internal:1234/v1",
        "code-review-mcp:test"
      ]
    }
  }
}
```

The checked-in `ai-scanner-backend/.mcp.json` is a Windows example with machine-specific paths; adapt it for your OS and MCP host instead of using it unchanged. On Docker Desktop, `host.docker.internal` normally reaches LM Studio on the same computer. If the MCP host/container runs on a different machine, set `LMSTUDIO_BASE_URL` to an endpoint reachable from that machine. Do not use `host.docker.internal` from a Kubernetes pod. For a private repository, provide `GITHUB_TOKEN` using the MCP host's environment/secret facility; never pass it as a tool argument.

In the host, call `check_llm`, then `check_repo_access` for a public test repository. Start a `quick` review with `start_review`, poll using `get_review_status`, and retrieve the result using `get_report`. Confirm a JSON and Markdown report appear in the mounted reports directory. Use a repository you are authorized to review; never send a private repository token as an MCP tool argument.

This validates the standalone stdio MCP server. It is separate from the Kubernetes end-to-end test: Kubernetes reviews enter through the UI/gateway and run using the shared pipeline in `review-api` and `scanner-worker`.

## 12. Troubleshooting and cleanup

| Symptom | First check |
|---|---|
| `ImagePullBackOff` | Compare the pod image and tag with `sudo k3s ctr images ls -q`; re-import the exact tag if needed. |
| Pod cannot start due to a missing Secret | Check the pod Events, then verify the Secret names and keys from section 8. |
| `review-api` is not Ready | Inspect `/readyz`, the reports PVC mount, and writable `/tmp`. |
| `lmstudio` Service is missing or has no route | Check the Tailscale operator and the model machine's tailnet connectivity. |
| Review Job fails | Inspect `kubectl -n ai-scan-repos describe job <job-name>` and its pod logs; verify GitHub access, worker image tag, and model reachability. |
| GitHub login callback fails | Ensure the OAuth App callback and `GITHUB_OAUTH_REDIRECT_URI` match exactly. |

For advanced recovery, upgrades, image rebuilds, and teardown, see [DEPLOY-K3S.md](../ai-scanner-backend-deployment/DEPLOY-K3S.md). For the complete MCP tool reference, see [MCP Server Documentation](docs/15-mcp-server-documentation.md) and [mcp-server README](../ai-scanner-backend/mcp-server/README.md).