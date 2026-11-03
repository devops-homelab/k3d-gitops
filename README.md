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

### Dev Microservice
- App manifest: [argocd-apps/microservices/dev/application.yaml](argocd-apps/microservices/dev/application.yaml)
- Shared Helm chart: [charts/microservices/common](charts/microservices/common)
- Dev values: [charts/microservices/dev/values.yaml](charts/microservices/dev/values.yaml)

The dev app deploys `k3d-microservice` into `microservices-dev` using a pinned
Docker Hub image tag. Update `image.tag` in the dev values after publishing a new
image. Both health probes use `/health`.

Dev uses `imagePullSecrets: [{name: dockerhub-secret}]` for authenticated Docker
Hub pulls. Create this Secret in `microservices-dev` before syncing the app.
Use a Docker Hub access token with read-only permission, not your password.
GitHub Actions secrets are not automatically available to Kubernetes.

Run the following in Bash locally (the token is prompted without echoing it or
putting its literal value in shell history):

```bash
kubectl create namespace microservices-dev --dry-run=client -o yaml | kubectl apply -f -
read -r -s -p "Docker Hub read-only token: " DOCKERHUB_TOKEN
printf '\n'
kubectl -n microservices-dev create secret docker-registry dockerhub-secret \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=navindujay \
  --docker-password="$DOCKERHUB_TOKEN" \
  --dry-run=client -o yaml | kubectl apply -f -
unset DOCKERHUB_TOKEN
```

Do not commit the token or an unencrypted Secret manifest. If managing the
credential through GitOps, use Sealed Secrets. The shared chart defaults to
`imagePullSecrets: []`, so other deployments can still pull public images without
authentication.

Ingress is disabled by default in the shared chart. Dev enables it with the
`traefik` class and routes `http://microservice.localhost/` to the service on port
8080. The chart supports ingress annotations, multiple hosts/paths, and TLS via
`ingress.tls` (a list of `secretName` and `hosts` entries); provision TLS secrets
separately.

Push these changes and sync `microservices-dev` in Argo CD. Traefik must be
installed, and its HTTP port must be exposed from k3d to your laptop. If you map
it to a port other than 80, include that port in the URL. If the hostname does
not resolve locally, map `microservice.localhost` to `127.0.0.1` in your hosts file.

Validate the dev chart locally:

```bash
helm lint charts/microservices/common -f charts/microservices/dev/values.yaml
helm template k3d-microservice charts/microservices/common \
  -n microservices-dev -f charts/microservices/dev/values.yaml
```

### Payment Tracker
- App manifest: `argocd-apps/microservices/dev/payment-tracker/application.yaml`
- Shared Helm chart: `charts/microservices/common`
- Dev values: `charts/microservices/dev/payment-tracker/values.yaml`

The payment tracker runs as `k3d-payment-tracker` in `microservices-dev` and is
served at `http://payment-tracker.localhost/` through Traefik. It reuses the
namespace's `dockerhub-secret` image pull credential and exposes `/healthz` for
readiness and liveness probes. SQLite data is stored in the `local-path` PVC
`k3d-payment-tracker-data`, which Argo CD is configured not to prune. Argo CD
discovers the application through the recursive root app configuration.

When moving from the old browser-only release, open the new image once at the
same local origin used before (for example, `http://payment-tracker.localhost:8088/`)
to migrate non-demo browser entries into SQLite before switching to the
Tailscale hostname. Browser storage is isolated by origin.

The payment-tracker repository's GitHub Actions workflow publishes the image and
updates its pinned tag in the dev values file. Validate the chart with:

```bash
helm lint charts/microservices/common \
	-f charts/microservices/dev/payment-tracker/values.yaml
helm template k3d-payment-tracker charts/microservices/common \
	-n microservices-dev -f charts/microservices/dev/payment-tracker/values.yaml
```

### Private Tailscale Access
The Tailscale Kubernetes Operator is installed by the Argo CD Application at
`argocd-apps/infra/tailscale-operator/application.yaml`, pinned to chart
`1.104.4`. It exposes the payment tracker only to your tailnet through the
`tailscale` Ingress class; do not enable Tailscale Funnel for this app.

Before pushing the GitOps changes, complete these one-time Tailscale setup
steps:

1. Enable MagicDNS and HTTPS certificates in your Tailscale admin console.
2. Create an OAuth client with Services read/write, Devices Core read/write,
	 and Auth Keys read/write scopes. Assign it `tag:k8s-operator`.
3. Merge the `tagOwners` entries for `tag:k8s-operator` and `tag:k8s`, and the
	 `autoApprovers.services` entry for `tag:k8s`, from Tailscale's Kubernetes
	 Operator quickstart into your existing tailnet ACL policy.
4. Seal the OAuth credentials into the resource path consumed by the Argo app.
	 Enter the client ID and secret only at the terminal prompts; never commit a
	 plaintext Secret:

```zsh
set -o pipefail
read -r "TS_CLIENT_ID?Tailscale OAuth client ID: "
read -rs "TS_CLIENT_SECRET?Tailscale OAuth client secret: "
printf '\n'

tmp_credentials=$(mktemp)
chmod 600 "$tmp_credentials"
printf 'client_id=%s\nclient_secret=%s\n' \
	"$TS_CLIENT_ID" "$TS_CLIENT_SECRET" > "$tmp_credentials"
unset TS_CLIENT_ID TS_CLIENT_SECRET

sealed_file=argocd-apps/infra/tailscale-operator/resources/operator-oauth-sealedsecret.yaml
seal_status=0
if kubectl create secret generic operator-oauth -n tailscale \
	--from-env-file="$tmp_credentials" \
	--dry-run=client -o yaml |
kubeseal --controller-name sealed-secrets-operator \
	--controller-namespace sealed-secrets \
	--name operator-oauth --namespace tailscale --format yaml \
	> "$sealed_file"; then
	printf 'Created %s\n' "$sealed_file"
else
	seal_status=$?
	rm -f "$sealed_file"
	printf 'Sealing failed; no manifest was kept.\n' >&2
fi
rm -f "$tmp_credentials"
if [[ $seal_status -ne 0 ]]; then
	false
fi
```

The operator app is discovered by the existing root app. Once synced, Tailscale
will provision a private HTTPS hostname for the app. Find it with:

```bash
kubectl -n microservices-dev get ingress k3d-payment-tracker-tailnet
```

Install and sign in to Tailscale on the iPhone using the same tailnet, then use
the hostname shown in the Ingress `ADDRESS` field. The `payment-tracker` TLS
host requests a name like `https://payment-tracker.<your-tailnet>.ts.net/`.

### Payment Tracker Authentication
The browser UI and transaction/recurring APIs require HTTP Basic Auth. The
`/api/import/ntb` Shortcut endpoint uses a separate bearer token; Kubernetes
health probes remain unauthenticated. Store `app-username`, `app-password`, and
`api-token` in the `payment-tracker-api` SealedSecret at
`charts/microservices/dev/payment-tracker/resources/payment-tracker-api-sealedsecret.yaml`.
The Argo app includes this resource directory. Use a unique random password for
the browser and a different strong random token in the iPhone Shortcut's
Authorization header. The API parses the supported NTB debit message but stores
only the merchant, amount, date, and category; it does not store the SMS,
account number, or current balance. Identical imports are deduplicated,
including the existing LKR 29,856.00 life-insurance entry.

Create the SealedSecret manifest from zsh without putting credentials in shell
history or passing them as process arguments:

```zsh
set -o pipefail
read -r "DAYBOOK_USERNAME?Daybook browser username: "
read -rs "DAYBOOK_PASSWORD?Daybook browser password: "
read -rs "DAYBOOK_API_TOKEN?Daybook Shortcut bearer token: "
printf '\n'

tmp_credentials=$(mktemp)
chmod 600 "$tmp_credentials"
printf 'app-username=%s\napp-password=%s\napi-token=%s\n' \
	"$DAYBOOK_USERNAME" "$DAYBOOK_PASSWORD" "$DAYBOOK_API_TOKEN" > "$tmp_credentials"
unset DAYBOOK_USERNAME DAYBOOK_PASSWORD DAYBOOK_API_TOKEN

sealed_file=charts/microservices/dev/payment-tracker/resources/payment-tracker-api-sealedsecret.yaml
seal_status=0
kubectl create secret generic payment-tracker-api -n microservices-dev \
	--from-env-file="$tmp_credentials" \
	--dry-run=client -o yaml |
kubeseal --controller-name sealed-secrets-operator \
	--controller-namespace sealed-secrets \
	--name payment-tracker-api --namespace microservices-dev --format yaml \
	> "$sealed_file"
seal_status=$?
rm -f "$tmp_credentials"
if [[ $seal_status -ne 0 ]]; then
	rm -f "$sealed_file"
	printf 'Sealing failed; no manifest was kept.\n' >&2
	false
else
	printf 'Created %s\n' "$sealed_file"
fi
```

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
