# Using the Onboard New App to OpenShift with TLS (cert-manager) Template

Use this guide to onboard a new application into OpenShift with TLS certificates issued through cert-manager.

## What This Template Creates

When you run the template, it scaffolds and publishes a repo that includes:

- Kubernetes namespace and deployment manifests
- cert-manager Certificate configuration
- OpenShift Route with TLS
- ArgoCD application registration (if configured)
- Catalog registration for the new component

## Before You Start

Make sure you have:

- Access to RHDH and the Create page
- A valid repo destination (currently GitHub in this template)
- A valid owner Group and system entity in the catalog
- A hostname that resolves for your cluster domain
- A ClusterIssuer available in the cluster (for example `letsencrypt-prod`, `letsencrypt-staging`, or `internal-ca`)

## How To Run The Template

1. In RHDH, go to Create.
2. Select Onboard New App to OpenShift with TLS (cert-manager).
3. Complete the Application Details section:
   - Application Name: lowercase, alphanumeric, and hyphens only
   - Description: short purpose statement
   - Owner Team: select an existing Group
   - System: select an existing System
4. Complete the OpenShift Configuration section:
   - Target Namespace
   - Environment (`dev`, `test`, `staging`, `prod`)
   - Replicas
   - Container image and port
   - Optional CPU and memory limits
5. Complete the TLS / cert-manager section:
   - ClusterIssuer
   - Hostname
   - TLS Secret Name (or leave blank for auto-generated value)
6. Choose Source Control destination.
7. Review and click Create.

## What To Verify After Creation

- The repository was created and contains generated manifests
- The entity appears in the Software Catalog
- ArgoCD app was created (if enabled)
- Certificate object is issued in the target namespace
- Route is reachable over HTTPS at the selected hostname

## Troubleshooting

### Template action errors

If you see errors like `publish:github is not registered` or `argocd:create-resources is not registered`, enable the required scaffolder backend modules in your RHDH dynamic plugins config.

### Owner or system relation warnings

If catalog warns about unresolved entities, ensure the selected Group and System exist and are registered in the catalog.

### TLS certificate not becoming Ready

Check:

- ClusterIssuer status
- DNS for hostname
- cert-manager logs/events

### Route not serving TLS

Check:

- Certificate secret exists in the namespace
- Route references the expected TLS secret
- Application pod and service are healthy

## Suggested Demo Input Values

- App Name: `demo-app`
- Namespace: `demo-app-dev`
- Environment: `dev`
- Image: `quay.io/redhat-developer/developer-images:latest`
- Port: `8080`
- ClusterIssuer: `letsencrypt-staging`
- Hostname: `demo-app.apps.<cluster-domain>`
