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

The runner scale set enables `containerMode.type: dind`. ARC adds a Docker-in-Docker
daemon and configures the runner's Docker connection so workflows can use Docker
Buildx to build and push images. This uses a privileged container; run only trusted
workflows on this runner scale set.

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

### Docker Buildx cannot connect to /var/run/docker.sock
The runner needs a Docker daemon, not just the Docker CLI. Keep
`containerMode.type: dind` in the runner scale set values and sync the
`gha-runner-scale-set` Argo CD application after pushing the change.
Wait for replacement runner pods before rerunning the failed workflow.
The cluster must allow privileged containers in `arc-runners`.

To check that Docker is available in a replacement runner:

```bash
kubectl -n arc-runners get pods
kubectl -n arc-runners exec <runner-pod> -c runner -- docker info
```

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
