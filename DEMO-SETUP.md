# Red Hat Developer Hub Demo – Customer Onboarding with cert-manager on OpenShift

**Demo scenario:** A developer uses Red Hat Developer Hub (RHDH) to onboard a new application onto OpenShift. The Software Template automatically provisions the namespace, deploys the app, requests a signed TLS certificate via cert-manager, and exposes the app over HTTPS through an OpenShift Route — all from a single self-service wizard.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Cluster Preparation](#2-cluster-preparation)
3. [Install Required Operators](#3-install-required-operators)
4. [Configure OpenShift GitOps (ArgoCD)](#4-configure-openshift-gitops-argocd)
5. [Configure OpenShift Pipelines (Tekton)](#5-configure-openshift-pipelines-tekton)
6. [Configure cert-manager](#6-configure-cert-manager)
7. [Configure Red Hat Developer Hub](#7-configure-red-hat-developer-hub)
8. [Load the Software Template](#8-load-the-software-template)
9. [Pre-Demo Checklist](#9-pre-demo-checklist)
10. [Running the Demo](#10-running-the-demo)
11. [Teardown](#11-teardown)
12. [Troubleshooting](#12-troubleshooting)

---

## 1. Prerequisites

### Tooling (on your laptop / jump host)

| Tool | Minimum version | Install |
|------|----------------|---------|
| `oc` CLI | 4.14 | [mirror.openshift.com](https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/) |
| `helm` | 3.13 | `brew install helm` / [helm.sh](https://helm.sh/docs/intro/install/) |
| `git` | 2.40 | OS package manager |
| `curl` / `jq` | any | OS package manager |

### OpenShift Cluster

- **Version:** OpenShift Container Platform 4.14 or later
- **Nodes:** Minimum 3 worker nodes (4 vCPU / 16 GB RAM each recommended)
- **Storage:** A default `StorageClass` that supports `ReadWriteOnce` (RWO) PVCs
- **DNS:** A wildcard DNS entry `*.apps.<cluster-domain>` resolving to the OpenShift router
- **Internet access:** Required for Let's Encrypt ACME challenges and pulling operator images (or configure a mirror registry)

### Accounts / Tokens needed

| Account | Purpose |
|---------|---------|
| OpenShift `cluster-admin` | Install operators and configure cluster |
| GitHub / GitLab personal access token (PAT) | RHDH scaffolding will push generated repos |
| (Optional) Let's Encrypt email address | Registered in the ClusterIssuer |

---

## 2. Cluster Preparation

### 2.1 Log in to the cluster

```bash
oc login --token=<your-token> --server=https://api.<cluster-domain>:6443
# Verify cluster-admin
oc whoami
oc auth can-i '*' '*' --all-namespaces
```

### 2.2 Confirm the default StorageClass

```bash
oc get storageclass
# One entry must have   ANNOTATIONS   storageclass.kubernetes.io/is-default-class=true
```

If none is marked default:

```bash
oc patch storageclass <name> \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

### 2.3 Create namespaces used throughout the demo

```bash
oc new-project cert-manager
oc new-project openshift-gitops    # created automatically by the GitOps operator; skip if it already exists
oc new-project rhdh                # Red Hat Developer Hub instance
```

### 2.4 Set up a Git organization / group for scaffolded repos

1. Create a GitHub organization (e.g. `rhdh-demo-org`) or a GitLab group.
2. Generate a PAT with scopes: `repo`, `workflow`, `admin:org` (GitHub) or `api` (GitLab).
3. Store the token — you will need it in [Section 7](#7-configure-red-hat-developer-hub).

---

## 3. Install Required Operators

All operators below are installed via OperatorHub through the OpenShift web console **or** via the CLI manifests shown. Install them in order.

### 3.1 cert-manager Operator for Red Hat OpenShift

**Web console path:** Operators → OperatorHub → search *"cert-manager"* → select **cert-manager Operator for Red Hat OpenShift** (Red Hat certified) → Install

**CLI alternative:**

```bash
cat <<'EOF' | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-cert-manager-operator
  namespace: openshift-operators
spec:
  channel: stable-v1
  name: openshift-cert-manager-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  installPlanApproval: Automatic
EOF
```

Wait for the operator pod to become ready:

```bash
oc wait deployment/cert-manager-operator-controller-manager \
  --for=condition=Available \
  --timeout=120s \
  -n cert-manager-operator
```

Verify cert-manager components are running in their dedicated namespace:

```bash
oc get pods -n cert-manager
# Expected: cert-manager, cert-manager-cainjector, cert-manager-webhook  →  Running
```

### 3.2 Red Hat OpenShift GitOps (ArgoCD)

**Web console path:** Operators → OperatorHub → search *"OpenShift GitOps"* → Install (All namespaces)

**CLI alternative:**

```bash
cat <<'EOF' | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-gitops-operator
  namespace: openshift-operators
spec:
  channel: latest
  name: openshift-gitops-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  installPlanApproval: Automatic
EOF
```

Wait:

```bash
oc wait deployment/openshift-gitops-server \
  --for=condition=Available \
  --timeout=180s \
  -n openshift-gitops
```

### 3.3 Red Hat OpenShift Pipelines (Tekton)

**Web console path:** Operators → OperatorHub → search *"OpenShift Pipelines"* → Install (All namespaces)

**CLI alternative:**

```bash
cat <<'EOF' | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-pipelines-operator-rh
  namespace: openshift-operators
spec:
  channel: latest
  name: openshift-pipelines-operator-rh
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  installPlanApproval: Automatic
EOF
```

Wait:

```bash
oc wait deployment/tekton-pipelines-controller \
  --for=condition=Available \
  --timeout=180s \
  -n openshift-pipelines
```

### 3.4 Red Hat Developer Hub (RHDH)

**Web console path:** Operators → OperatorHub → search *"Red Hat Developer Hub"* → Install (namespace: `rhdh`)

**CLI alternative:**

```bash
# Create the OperatorGroup first (namespace-scoped install)
cat <<'EOF' | oc apply -f -
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: rhdh-operator-group
  namespace: rhdh
spec:
  targetNamespaces:
    - rhdh
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: rhdh
  namespace: rhdh
spec:
  channel: fast
  name: rhdh
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  installPlanApproval: Automatic
EOF
```

Confirm the operator CSV is ready before proceeding:

```bash
oc get csv -n rhdh | grep rhdh
# STATUS should be: Succeeded
```

---

## 4. Configure OpenShift GitOps (ArgoCD)

### 4.1 Grant ArgoCD permission to manage the demo namespaces

```bash
# Allow the ArgoCD service account to manage resources cluster-wide
oc adm policy add-cluster-role-to-user cluster-admin \
  -z openshift-gitops-argocd-application-controller \
  -n openshift-gitops
```

### 4.2 Create an ArgoCD Project for each environment

```bash
for ENV in dev test staging prod; do
cat <<EOF | oc apply -f -
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: ${ENV}
  namespace: openshift-gitops
spec:
  description: "RHDH demo – ${ENV} environment"
  sourceRepos:
    - '*'
  destinations:
    - namespace: '*'
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: '*'
      kind: '*'
EOF
done
```

### 4.3 Retrieve the ArgoCD admin password (for demo use)

```bash
oc extract secret/openshift-gitops-cluster \
  -n openshift-gitops --to=-
# Or open the ArgoCD UI:
oc get route openshift-gitops-server -n openshift-gitops -o jsonpath='{.spec.host}'
```

---

## 5. Configure OpenShift Pipelines (Tekton)

### 5.1 Install the Tekton Hub cluster tasks used by the Pipeline

The demo pipeline uses four pre-built, reusable Tasks. In OpenShift Pipelines 1.14+ (Tekton v0.50+), the `ClusterTask` API was removed. Tasks are now namespace-scoped objects installed into the `openshift-pipelines` namespace and referenced by the **cluster resolver**. This section explains how to verify they are present and how to fill any gaps.

#### Background — what each Task does

| Task name | Purpose in the pipeline |
|-----------|------------------------|
| `git-clone` | Clones the application source repository into the shared workspace so later steps can build and test the code. |
| `buildah` | Builds a container image from the `Dockerfile` in the cloned repo and pushes it to the target registry (Quay.io, etc.) using Buildah — no Docker daemon required. |
| `openshift-client` | Runs `oc` commands inside the cluster; used to apply the namespace, Deployment, cert-manager Certificate, and OpenShift Route manifests. |
| `curl` | Issues an HTTPS request to the live Route as a post-deployment smoke test, verifying the TLS endpoint is reachable before the pipeline is marked successful. |

#### Step 1 — Check which Tasks are already present

> **Note:** `ClusterTask` was removed in OpenShift Pipelines 1.14 (Tekton Pipelines v0.50). Tasks are now namespace-scoped objects installed into the `openshift-pipelines` namespace and referenced via the **cluster resolver**. There is no `clustertask` resource type on modern clusters.

Verify the Tasks shipped by the Pipelines operator:

```bash
oc get task -n openshift-pipelines | grep -E 'git-clone|buildah|openshift-client|curl'
```

Expected output (versions may vary):

```
buildah                   20d
git-clone                 20d
openshift-client          20d
```

> `curl` is not bundled with the operator. It is installed separately in Step 3 below.

#### Step 2 — Install the `tkn` CLI (if not already present)

The `tkn` CLI is used to interact with Tekton and install tasks from Tekton Hub. There is no official apt/deb package — install the binary directly from the GitHub release.

**Debian / Ubuntu (including WSL):**

```bash
# Set the version you want to install
TKN_VERSION=0.37.0

# Download the tarball for Linux amd64
curl -LO "https://github.com/tektoncd/cli/releases/download/v${TKN_VERSION}/tkn_${TKN_VERSION}_Linux_x86_64.tar.gz"

# Extract only the tkn binary
tar xzf "tkn_${TKN_VERSION}_Linux_x86_64.tar.gz" tkn

# Move it onto your PATH
sudo mv tkn /usr/local/bin/tkn
sudo chmod +x /usr/local/bin/tkn

# Clean up
rm "tkn_${TKN_VERSION}_Linux_x86_64.tar.gz"
```

**RHEL / Fedora:**

```bash
sudo dnf install tektoncd-cli
```

**macOS:**

```bash
brew install tektoncd-cli
```

Verify the installation:

```bash
tkn version
# Should print the tkn client version and the Tekton Pipelines version on the cluster
```

> For the latest release number, check the [Tekton CLI releases page](https://github.com/tektoncd/cli/releases).

#### Step 3 — Install any missing Tasks

> **Note:** The original Tekton Hub (`api.hub.tekton.dev`) has been **decommissioned**. `tkn hub install` will fail with a DNS resolution error even with internet access. Use one of the methods below instead.

**Option A — direct `oc apply` from raw GitHub (recommended — requires access to raw.githubusercontent.com):**

```bash
BASE=https://raw.githubusercontent.com/tektoncd/catalog/main/task

oc apply -f ${BASE}/git-clone/0.9/git-clone.yaml  -n openshift-pipelines
oc apply -f ${BASE}/buildah/0.6/buildah.yaml       -n openshift-pipelines
oc apply -f ${BASE}/curl/0.1/curl.yaml             -n openshift-pipelines
```

**Option B — fully offline / air-gapped (no external internet required):**

Apply the `curl` Task inline. `git-clone`, `buildah`, and `openshift-client` are bundled with the Pipelines operator and should already be present from Step 1.

```bash
cat <<'EOF' | oc apply -f -
apiVersion: tekton.dev/v1
kind: Task
metadata:
  name: curl
  namespace: openshift-pipelines
  labels:
    app.kubernetes.io/version: "0.1"
spec:
  description: Issues an HTTP/HTTPS request using curl.
  params:
    - name: url
      description: The URL to request
      type: string
    - name: options
      description: Additional curl flags (e.g. --fail --retry 5)
      type: string
      default: "--fail --silent --show-error"
  steps:
    - name: curl
      image: registry.access.redhat.com/ubi9/ubi-minimal:latest
      script: |
        #!/bin/sh
        set -e
        curl $(params.options) "$(params.url)"
EOF
```

Verify all four Tasks are now available:

```bash
oc get task -n openshift-pipelines | grep -E 'git-clone|buildah|openshift-client|curl'
# All four should appear before continuing
```

#### Step 4 — Understand the cluster resolver reference in the pipeline

The pipeline manifest (`skeleton/tekton/pipeline.yaml`) references Tasks using the **cluster resolver**, which looks up a ClusterTask by name:

```yaml
taskRef:
  resolver: cluster
  params:
    - name: kind
      value: Task
    - name: name
      value: git-clone
    - name: namespace
      value: openshift-pipelines
```

This means the Tasks must exist as namespace-scoped `Task` objects in the `openshift-pipelines` namespace. The `ClusterTask` API no longer exists on OpenShift Pipelines 1.14+; the cluster resolver is the supported replacement.

#### Step 5 — Confirm the Tekton feature flags allow the cluster resolver

```bash
oc get configmap feature-flags -n openshift-pipelines -o yaml | grep enable-cluster-resolver
# Should show:  enable-cluster-resolver: "true"
```

If the flag is missing or set to `false`, enable it:

```bash
oc patch configmap feature-flags \
  -n openshift-pipelines \
  --type merge \
  -p '{"data":{"enable-cluster-resolver":"true"}}'
```

### 5.2 Create a pipeline service account and registry secret

> **Red Hat SSO users:** Quay.io uses Red Hat SSO for interactive logins, but the SSO flow is browser-based and **cannot be used directly in scripts**. You must obtain a non-SSO credential first using one of the two methods below before running any `oc create secret` command.

#### Method 1 — Quay.io encrypted CLI password

> **Known issue for Red Hat SSO users:** The "re-enter your password" prompt on the Generate Encrypted Password page expects a standalone Red Hat account password. Red Hat SSO accounts federated through `sso.redhat.com` typically have no standalone password set, so this step will fail regardless of what PIN or token you enter. **Skip this method and use Method 2 or 3 instead.**

#### Method 2 — Quay.io robot account (recommended for shared / persistent demos)

Robot accounts are service identities that are not tied to a personal SSO session and are the preferred approach for pipelines.

1. Log into [quay.io](https://quay.io) and open your organisation or personal namespace.
2. Create the target repository first (for example, `demo-app`):
  - Repositories → **Create New Repository**
  - Name: `demo-app`
  - Visibility: `Private` (recommended)
3. Navigate to **Robot Accounts** → **Create Robot Account**.
4. Give it a name (e.g. `rhdh-demo-pipeline`).
5. Open the repository you created (`demo-app`) → **Settings** → **Permissions** (or **Repository Permissions**), add robot account `<namespace>+rhdh-demo-pipeline`, and grant:
  - `Write` to push images
  - `Read` to pull images
6. Click the robot account name → **Robot Token** to reveal/copy the token.

Use the robot account credentials in the secret:

```bash
# Robot account username format is:  <namespace>+<robot-name>
oc create secret docker-registry registry-credentials \
  --docker-server=quay.io \
  --docker-username=<quay-namespace>+rhdh-demo-pipeline \
  --docker-password=<copied-robot-token> \
  --docker-email=<your-rh-email> \
  -n demo-app-dev
```

Example (replace with your values):

```bash
oc create secret docker-registry registry-credentials \
  --docker-server=quay.io \
  --docker-username=myteam+rhdh-demo-pipeline \
  --docker-password=<copied-robot-token> \
  --docker-email=you@redhat.com \
  -n demo-app-dev
```

#### Method 3 — `podman login` (quickest for personal use, works with Red Hat SSO)

`podman login` supports the full Red Hat SSO browser flow and saves credentials to a local auth file that can be imported directly as an OpenShift secret.

```bash
# Log in interactively — opens SSO browser flow if needed
podman login quay.io

# The credentials are saved to:
#   Podman:  ${XDG_RUNTIME_DIR}/containers/auth.json  (Linux)
#   Docker:  ~/.docker/config.json  (if DOCKER_CONFIG is set or docker is used)

# Confirm the auth file path
podman info --format '{{.Store.GraphRoot}}' 2>/dev/null || true
AUTH_FILE="${XDG_RUNTIME_DIR}/containers/auth.json"
[ -f "$AUTH_FILE" ] || AUTH_FILE="$HOME/.docker/config.json"
echo "Using auth file: $AUTH_FILE"

# Create the pull secret from the auth file
oc create secret generic registry-credentials \
  --from-file=.dockerconfigjson="$AUTH_FILE" \
  --type=kubernetes.io/dockerconfigjson \
  -n <target-namespace>
```

> If `podman` is not installed: `sudo dnf install podman` (RHEL/Fedora) or `brew install podman` (macOS).

#### Link the secret to the pipeline service account

Run this after any method above (Method 1, 2, or 3):

```bash
oc secrets link pipeline registry-credentials --for=pull,mount -n <target-namespace>
```

Verify the secret is linked:

```bash
oc get serviceaccount pipeline -n <target-namespace> -o jsonpath='{.secrets}' | jq .
```

---

## 6. Configure cert-manager

### 6.1 Apply the ClusterIssuers

The ClusterIssuer definitions are in the skeleton manifest and need to be applied once to the cluster by a platform admin.

```bash
# Update the Let's Encrypt email address before applying
sed -i 's/platform-engineering@example.com/brentjon@redhat.com/' \
  skeleton/manifests/clusterissuers.yaml

oc apply -f skeleton/manifests/clusterissuers.yaml
```

Verify:

```bash
oc get clusterissuer
# NAME                    READY   AGE
# letsencrypt-prod        True    ...
# letsencrypt-staging     True    ...
# internal-ca             True    ...   ← only if the CA Secret exists
```

### 6.2 (Optional) Set up the Internal CA ClusterIssuer

If you will use the **internal-ca** issuer during the demo:

```bash
# Generate a self-signed CA (demo only — use your real corporate CA in production)
openssl req -x509 -newkey rsa:4096 -keyout ca.key -out ca.crt \
  -days 3650 -nodes -subj "/CN=Demo Internal CA/O=Example Corp"

oc create secret tls internal-ca-key-pair \
  --cert=ca.crt --key=ca.key \
  -n cert-manager
```

Trust the CA in your browser / OS before the demo so HTTPS connections show as valid.

### 6.3 Enable cert-manager OpenShift Routes integration

This allows cert-manager to inject the TLS certificate directly into an OpenShift Route:

```bash
cat <<'EOF' | oc apply -f -
apiVersion: operator.openshift.io/v1alpha1
kind: CertManager
metadata:
  name: cluster
spec:
  logLevel: Normal
  operatorLogLevel: Normal
  unsupportedConfigOverrides:
    controller:
      args:
        - --enable-certificate-owner-ref=true
        - --feature-gates=ExperimentalGatewayAPISupport=false
EOF
```

Verify cert-manager is healthy (there is no separate routes pod in this operator build):

```bash
oc get deployment -n cert-manager cert-manager cert-manager-cainjector cert-manager-webhook
```

Then verify Route + Certificate integration functionally during deployment:

```bash
oc get certificate -n <target-namespace>
oc get route <app-name> -n <target-namespace>
```

---

## 7. Configure Red Hat Developer Hub

### 7.1 Create configuration secrets

```bash
# GitHub integration (replace with GitLab/Bitbucket values if needed)
oc create secret generic rhdh-github-secret \
  --from-literal=GITHUB_TOKEN=<your-github-pat> \
  --from-literal=GITHUB_ORG=<your-github-org> \
  -n rhdh

# ArgoCD integration credentials
ARGOCD_PASSWORD=$(oc extract secret/openshift-gitops-cluster \
  -n openshift-gitops --to=- 2>/dev/null | tr -d '\n')

oc create secret generic rhdh-argocd-secret \
  --from-literal=ARGOCD_SERVER=$(oc get route openshift-gitops-server \
    -n openshift-gitops -o jsonpath='{.spec.host}') \
  --from-literal=ARGOCD_USER=admin \
  --from-literal=ARGOCD_PASSWORD="${ARGOCD_PASSWORD}" \
  -n rhdh
```

### 7.2 Create the RHDH `app-config` ConfigMap

```bash
cat <<'EOF' | oc apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: rhdh-app-config
  namespace: rhdh
data:
  app-config.yaml: |
    app:
      title: "Red Hat Developer Hub – TLS Demo"
      baseUrl: https://rhdh.apps.xps.planenotsimple.com

    organization:
      name: "Example Corp"

    backend:
      baseUrl: https://rhdh.apps.xps.planenotsimple.com
      cors:
        origin: https://rhdh.apps.xps.planenotsimple.com

    integrations:
      github:
        - host: github.com
          token: ${GITHUB_TOKEN}

    catalog:
      locations:
        # Register the Software Template
        - type: url
          target: https://github.com/brentnjones/duke/blob/main/template.yaml
          rules:
            - allow: [Template]

    scaffolder:
      defaultAuthor:
        name: RHDH Scaffolder
        email: rhdh@example.com

    argocd:
      username: ${ARGOCD_USER}
      password: ${ARGOCD_PASSWORD}
      appLocatorMethods:
        - type: config
          instances:
            - name: main
              url: https://${ARGOCD_SERVER}

    kubernetes:
      serviceLocatorMethod:
        type: multiTenant
      clusterLocatorMethods:
        - type: config
          clusters:
            - name: local
              url: https://kubernetes.default.svc
              authProvider: serviceAccount
              skipTLSVerify: false
EOF
```

> **Note:** Replace `<cluster-domain>` and `<your-github-org>` with your actual values.

### 7.3 Create the RHDH instance

```bash
cat <<'EOF' | oc apply -f -
apiVersion: rhdh.redhat.com/v1alpha5
kind: Backstage
metadata:
  name: rhdh-demo
  namespace: rhdh
spec:
  application:
    appConfig:
      mountPath: /opt/app-root/src
      configMaps:
        - name: rhdh-app-config
    extraEnvs:
      secrets:
        - name: rhdh-github-secret
        - name: rhdh-argocd-secret
    replicas: 1
    route:
      enabled: true
      tls:
        enabled: true
        termination: edge
EOF
```

Wait for RHDH to be ready:

```bash
oc wait deployment/backstage-rhdh-demo \
  --for=condition=Available \
  --timeout=300s \
  -n rhdh

# Get the RHDH URL
oc get route backstage-rhdh-demo -n rhdh -o jsonpath='{.spec.host}'
```

### 7.4 Grant RHDH Kubernetes read access

```bash
oc adm policy add-cluster-role-to-user view \
  -z rhdh-demo \
  -n rhdh
```

---

## 8. Load the Software Template

### 8.1 Create a repository (if you do not have one yet) and push the template

If you do not already have a repository, create one first.

Option A - GitHub CLI:

```bash
# Authenticate GH CLI first (one-time)
gh auth login

# Create the repository in your org
# Use --private by default for customer demos
gh repo create <your-github-org>/duke --private --source=. --remote=origin --push
```

Option B - GitHub Web UI:

1. Go to https://github.com/new
2. Set Owner to your org, Repository name to duke
3. Choose visibility (see guidance below)
4. Create repository
5. Then run:

```bash
cd /home/brentjon/projects/Duke

git init
git add .
git commit -m "Initial commit - RHDH cert-manager demo template"
git branch -M main
git remote add origin https://github.com/<your-github-org>/duke.git
git push -u origin main
```

Repository visibility guidance:

- Private (recommended): best for customer demos and internal platform assets. Use this when RHDH has a GitHub token configured (Section 7.1).
- Public: only choose this if you intentionally want anyone to view the template repo or if you do not want to configure authentication for catalog/template reads.

For your use case, choose private.

### 8.2 Verify the template appears in RHDH

1. Open the RHDH URL in a browser.
2. Navigate to **Create** (the "+" icon in the sidebar).
3. Confirm **"Onboard New App to OpenShift with TLS (cert-manager)"** appears in the template catalog.

---

## 9. Pre-Demo Checklist

Run through this list 30 minutes before the customer arrives:

- [ ] `oc get clusterissuer` — all issuers show `READY: True`
- [ ] `oc get pods -n cert-manager` — all three pods `Running`
- [ ] `oc get pods -n openshift-gitops` — ArgoCD server `Running`
- [ ] `oc get pods -n openshift-pipelines` — Tekton controller `Running`
- [ ] RHDH URL loads in the browser and the Software Template is visible under **Create**
- [ ] GitHub PAT has not expired; test with `curl -H "Authorization: token <PAT>" https://api.github.com/user`
- [ ] DNS wildcard `*.apps.<cluster-domain>` resolves from your laptop
- [ ] (If using internal CA) CA certificate trusted in your demo browser profile
- [ ] Have the OpenShift web console open and logged in as a non-admin user to show the developer experience
- [ ] ArgoCD UI loaded and showing the `dev`/`test`/`staging`/`prod` projects

---

## 10. Running the Demo

### Step 1 — Open RHDH and start the wizard

1. Navigate to the RHDH URL.
2. In the left sidebar, click **Create** (rocket icon).
3. Click **Choose** on the *"Onboard New App to OpenShift with TLS (cert-manager)"* template.

> **Talking point:** "Instead of writing YAML, opening tickets, or waiting for platform team approval, developers fill in a simple form and everything is provisioned automatically."

---

### Step 2 — Fill in the Application Details form

| Field | Demo value |
|-------|-----------|
| Application Name | `demo-app` |
| Description | `Customer demo application with TLS` |
| Owner | `team-alpha` |
| System | `ecommerce-platform` |

Click **Next**.

---

### Step 3 — Fill in the OpenShift Configuration form

| Field | Demo value |
|-------|-----------|
| Target Namespace | `demo-app-dev` |
| Environment | `Development` |
| Replica Count | `1` |
| Container Image | `quay.io/redhat-developer/developer-images:latest` |
| Container Port | `8080` |
| CPU Limit | `500m` |
| Memory Limit | `512Mi` |

Click **Next**.

---

### Step 4 — Fill in the TLS / cert-manager Configuration form

| Field | Demo value |
|-------|-----------|
| ClusterIssuer | `Let's Encrypt (Staging / testing)` |
| Hostname | `demo-app.apps.<cluster-domain>` |
| TLS Secret Name | *(leave blank — auto-generated as `demo-app-tls`)* |

> **Talking point:** "The developer selects an issuer from a pre-approved list. The platform team controls which CAs are available. The developer never touches a certificate directly."

Click **Next**.

---

### Step 5 — Select the repository location

| Field | Demo value |
|-------|-----------|
| Repository Location | `github.com?owner=<your-github-org>&repo=demo-app` |

Click **Review** → **Create**.

---

### Step 6 — Watch the scaffolding steps execute

RHDH will display a live progress view:

1. **Fetch Application Skeleton** — copies the template files and substitutes values
2. **Publish to Source Control** — pushes the generated repo to GitHub
3. **Register App in ArgoCD / GitOps** — creates the ArgoCD Application
4. **Register in RHDH Software Catalog** — the component appears immediately

> **Talking point:** "In under 60 seconds we have a repo, a GitOps pipeline, and the app registered in the catalog — all without touching the cluster directly."

---

### Step 7 — Show GitOps sync in ArgoCD

1. Open the ArgoCD UI.
2. Find the `demo-app-dev` Application.
3. Show it syncing the manifests: Namespace → Deployment → Certificate → Route.

> **Talking point:** "ArgoCD continuously reconciles the cluster state with Git. If someone manually changes a resource, ArgoCD will revert it automatically — giving us a single source of truth."

---

### Step 8 — Show cert-manager issuing the certificate

```bash
# In a visible terminal during the demo:
watch oc get certificate -n demo-app-dev
```

Walk the audience through the state transitions:

| Status | What is happening |
|--------|------------------|
| `False / Pending` | cert-manager created the ACME order |
| `False / Awaiting` | HTTP-01 challenge pod is running |
| `True / Ready` | Certificate signed and stored in Secret `demo-app-tls` |

```bash
# Show the certificate details
oc get secret demo-app-tls -n demo-app-dev -o jsonpath='{.data.tls\.crt}' \
  | base64 -d | openssl x509 -noout -text | grep -E 'Subject:|DNS:|Not After'
```

> **Talking point:** "cert-manager automatically renews this certificate before it expires — no more manually tracking expiry dates or emergency rotations."

---

### Step 9 — Open the application over HTTPS

```bash
oc get route demo-app -n demo-app-dev -o jsonpath='{.spec.host}'
```

Open `https://demo-app.apps.<cluster-domain>` in the browser.

- Show the padlock / valid certificate in the browser.
- Show the HSTS header is set.

```bash
curl -I https://demo-app.apps.<cluster-domain>
# Look for:  Strict-Transport-Security: max-age=31536000;includeSubDomains;preload
```

---

### Step 10 — Return to RHDH and show the component page

1. Navigate to **Catalog** in RHDH.
2. Open the `demo-app` component.
3. Show the **Overview** tab: owner, system, description, live app link.
4. Show the **Kubernetes** tab: pod status, deployment, route — all without leaving RHDH.
5. Show the **CI/CD** tab: Tekton pipeline runs and ArgoCD sync status.

> **Talking point:** "The developer has a single pane of glass. They can see runtime health, TLS certificate status, and deployment history without needing access to the OpenShift console or ArgoCD directly."

---

## 11. Teardown

After the demo, clean up all resources:

```bash
# Delete the demo application namespace and all its resources
oc delete namespace demo-app-dev

# Remove the ArgoCD Application
oc delete application demo-app-dev -n openshift-gitops

# Remove the component from the RHDH catalog
# (via RHDH UI: Catalog → demo-app → ⋮ → Unregister Entity)

# Delete the generated GitHub repo (optional)
gh repo delete <your-github-org>/demo-app --yes
```

---

## 12. Troubleshooting

### cert-manager Certificate stuck in `Pending`

```bash
# Check the Certificate events
oc describe certificate demo-app-cert -n demo-app-dev

# Check the CertificateRequest
oc get certificaterequest -n demo-app-dev

# Check the ACME Challenge
oc get challenge -n demo-app-dev
oc describe challenge -n demo-app-dev

# Check cert-manager controller logs
oc logs -l app=cert-manager -n cert-manager --tail=50
```

Common causes:
- DNS for the hostname does not resolve publicly (use `letsencrypt-staging` or `internal-ca` for private clusters)
- The HTTP-01 challenge port (80) is blocked by a firewall
- The ClusterIssuer email is invalid

### Route not getting TLS / padlock not showing

```bash
oc describe route demo-app -n demo-app-dev
# Check:  TLS Termination, and that the Secret exists
oc get secret demo-app-tls -n demo-app-dev
```

### ArgoCD Application stuck in `OutOfSync`

```bash
oc describe application demo-app-dev -n openshift-gitops
```

Check that the ArgoCD service account has permission to create the resources in the target namespace.

### RHDH Template not visible in the catalog

```bash
# Check the catalog import logs
oc logs -l app.kubernetes.io/name=backstage -n rhdh --tail=100 | grep -i "template\|error"
```

Ensure the `template.yaml` URL in the `app-config.yaml` is publicly accessible (or that RHDH has credentials to access the repo).

### Tekton Pipeline fails on `apply-manifests` step

```bash
tkn pipelinerun logs -L -n demo-app-dev
```

Ensure the `pipeline` service account has the necessary RBAC to create `Certificate` and `Route` resources:

```bash
oc adm policy add-role-to-user edit \
  -z pipeline \
  -n demo-app-dev
```
