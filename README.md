# k3d-gitops

GitOps repository for a local k3d cluster managed by Argo CD.

This repo contains:
- Argo CD Application manifests under argocd-apps
- Helm values overlays under charts
- Infrastructure apps for Sealed Secrets and GitHub Actions Runner Controller (ARC)

## Repository Structure

```text
argocd-apps/
	infra/
		actions-runner-controller/
			application.yaml
		gha-runner-scale-set/
			application.yaml
			resources/
				github-auth-sealedsecret.yaml
		sealed-secrets/
			application.yaml
	microservices/

charts/
	infra/
		actions-runner-controller/
			values.yaml
		gha-runner-scale-set/
			values.yaml
		sealed-secrets/
			values.yaml
	microservices/
```

## Argo CD Apps

### Sealed Secrets
- App manifest: argocd-apps/infra/sealed-secrets/application.yaml
- Values: charts/infra/sealed-secrets/values.yaml

### ARC Controller
- App manifest: argocd-apps/infra/actions-runner-controller/application.yaml
- Chart: gha-runner-scale-set-controller
- Values: charts/infra/actions-runner-controller/values.yaml

Important sync options are enabled in the app to handle large CRDs:
- ServerSideApply=true
- Replace=true

### ARC Runner Scale Set
- App manifest: argocd-apps/infra/gha-runner-scale-set/application.yaml
- Chart: gha-runner-scale-set
- Values: charts/infra/gha-runner-scale-set/values.yaml
- Extra resource source: argocd-apps/infra/gha-runner-scale-set/resources

## GitHub App Authentication

The runner scale set uses:
- githubConfigSecret: github-auth

Secret keys required by ARC:
- github_app_id
- github_app_installation_id
- github_app_private_key

These values are stored as a SealedSecret:
- argocd-apps/infra/gha-runner-scale-set/resources/github-auth-sealedsecret.yaml

## Create or Rotate the Sealed Secret

Use your Sealed Secrets controller service details:
- controller name: sealed-secrets-operator
- controller namespace: sealed-secrets

Example command:

```bash
kubectl create secret generic github-auth \
	-n arc-runners \
	--from-literal=github_app_id="APP_ID" \
	--from-literal=github_app_installation_id="INSTALLATION_ID" \
	--from-file=github_app_private_key="./github-app-private-key.pem" \
	--dry-run=client -o yaml \
| kubeseal \
	--controller-name sealed-secrets-operator \
	--controller-namespace sealed-secrets \
	--format yaml \
> argocd-apps/infra/gha-runner-scale-set/resources/github-auth-sealedsecret.yaml
```

## Sync Order

For first-time bootstrap or after major changes:

1. sealed-secrets-operator
2. actions-runner-controller
3. gha-runner-scale-set

Example:

```bash
argocd app sync sealed-secrets-operator
argocd app sync actions-runner-controller
argocd app sync gha-runner-scale-set
```

## Validation Commands

```bash
kubectl get crd | grep actions.github.com
kubectl -n arc-runners get sealedsecret github-auth
kubectl -n arc-runners get secret github-auth
kubectl -n arc-runners get autoscalingrunnersets
kubectl -n arc-system logs deploy/actions-runner-controller-gha-rs-controller --tail=120
```

## Common Issues

### SharedResourceWarning for github-auth
If Argo reports that SealedSecret/github-auth is owned by multiple apps, ensure your root app-of-apps only includes application manifests (for example with directory include pattern **/application.yaml).

### CRD annotation too long
If sync fails on ARC CRDs with metadata.annotations too long, keep these sync options in actions-runner-controller app:
- ServerSideApply=true
- Replace=true

### 403 Resource not accessible by integration
This is a GitHub App permission/scope issue.

If githubConfigUrl is org-level (https://github.com/<org>):
- Grant org-level self-hosted runner permissions to the GitHub App (Read and write)
- Ensure app is installed for the organization

If githubConfigUrl is repo-level (https://github.com/<org>/<repo>):
- Ensure repository-level self-hosted runners are enabled
- Ensure app is installed on that repository

## Notes

- Do not commit plaintext GitHub App credentials.
- Committing SealedSecret manifests is expected and safe.

---

## Kokumi - Immutable Artifact Delivery

[Kokumi](https://kokumi.dev) provides immutable artifact delivery for Kubernetes, working alongside Argo CD.

### Key Concepts

- **Order**: Delivery request that triggers rendering of Helm charts or manifest bundles
- **Preparation**: Immutable OCI artifact produced from an Order (versioned, auditable)
- **Serving**: Active pointer to a Preparation that creates/updates an Argo CD Application
- **Menu**: Cluster-scoped reusable template with override policies

### Installation

1. Sync the Argo CD apps (fully GitOps-managed):

```bash
argocd app sync kokumi
argocd app sync kokumi-config
```

2. Verify installation:

```bash
kubectl get crds | grep kokumi.dev
kubectl get pods -n kokumi
```

### Access the Kokumi UI

```bash
kubectl port-forward -n kokumi svc/kokumi-server 8080:80
```

Open http://localhost:8080 (default: admin/admin)

### Example Order

Deploy a Helm chart via Kokumi:

```yaml
apiVersion: delivery.kokumi.dev/v1alpha1
kind: Order
metadata:
  name: my-app
spec:
  source:
    oci: oci://ghcr.io/stefanprodan/charts/podinfo
    version: "6.10.2"
  render:
    helm:
      namespace: default
      values:
        replicaCount: 2
  destination:
    oci: oci://kokumi-registry.kokumi.svc.cluster.local:5000/preparation/my-app
  autoDeploy: Enabled
```

### Kokumi Files

- App manifest: argocd-apps/infra/kokumi/application.yaml
- Registry credentials: argocd-apps/infra/kokumi/resources/argocd-registry-credentials.yaml
- Example Order: argocd-apps/infra/kokumi/examples/podinfo-order.yaml
- Example Menu: argocd-apps/infra/kokumi/examples/sealed-secrets-menu.yaml

### Sync Order (with Kokumi)

For first-time bootstrap:

1. sealed-secrets-operator
2. actions-runner-controller
3. gha-runner-scale-set
4. kokumi
5. kokumi-config