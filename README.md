# Argo CD App-of-Apps (Infra Test)

This repository is the GitOps source used by Argo CD App-of-Apps.


Both are managed by Argo CD Image Updater (GHCR `main` tag + digest write-back to Kustomize).

## Repository layout

- `bootstrap/`
  - `project.yaml`: Argo CD `AppProject`
  - `root-application.yaml`: root app that recursively manages `apps/`
  - `image-updater.yaml`: `ImageUpdater` CR
- `apps/<app-name>/application.yaml`
  - child Argo CD `Application` + image updater annotations
- `workloads/<app-name>/`
  - Kustomize workload manifests (`namespace`, `deployment`, `service`, `ingress`, `kustomization`)

## Bootstrap commands

Run once on cluster:

```bash
kubectl apply -f bootstrap/project.yaml
kubectl apply -f bootstrap/root-application.yaml
kubectl apply -f bootstrap/image-updater.yaml
```

Verify:

```bash
kubectl get app -n argocd
kubectl get imageupdater -n argocd
```

## Add a new application (standard workflow)

Example app name: `my-app`

### 1) Add child Argo app

Create:
- `apps/my-app/application.yaml`

Use this template:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
  annotations:
    argocd-image-updater.argoproj.io/image-list: app=ghcr.io/<owner>/my-app:main
    argocd-image-updater.argoproj.io/app.update-strategy: digest
    argocd-image-updater.argoproj.io/app.allow-tags: regexp:^main$
    argocd-image-updater.argoproj.io/app.kustomize.image-name: ghcr.io/<owner>/my-app
    argocd-image-updater.argoproj.io/write-back-method: git
    argocd-image-updater.argoproj.io/write-back-target: kustomization
    argocd-image-updater.argoproj.io/git-branch: main
spec:
  project: platform-apps
  source:
    repoURL: https://github.com/zubeyranwar/App-of-Apps.git
    targetRevision: main
    path: workloads/my-app
  destination:
    server: https://kubernetes.default.svc
    namespace: my-app
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### 2) Add workload manifests

Create folder:
- `workloads/my-app/`

Required files:
- `namespace.yaml`
- `deployment.yaml`
- `service.yaml`
- `ingress.yaml`
- `kustomization.yaml`

Minimum `kustomization.yaml` pattern:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: my-app
resources:
  - namespace.yaml
  - deployment.yaml
  - service.yaml
  - ingress.yaml
images:
  - name: ghcr.io/<owner>/my-app
    newTag: main
    newName: ghcr.io/<owner>/my-app
```

### 3) Push changes

```bash
git add apps/my-app workloads/my-app
git commit -m "Add my-app to App-of-Apps"
git push
```

### 4) Force refresh (optional, speeds up)

```bash
kubectl annotate app platform-app-of-apps -n argocd argocd.argoproj.io/refresh=hard --overwrite
kubectl get app my-app -n argocd
```

## App repository requirements (for auto image updates)

Each app repo should have:
- `Dockerfile`
- `.github/workflows/publish-ghcr.yml` that pushes `ghcr.io/<owner>/<app>:main`

## Operational checks

```bash
kubectl get app -n argocd
kubectl get pods -A | grep -E 'argocd|ingress-nginx'
kubectl logs -n argocd deploy/argocd-image-updater-controller --since=10m | tail -n 100
```

## Local testing without DNS (self-healing)

Use this when public DNS/ingress is not ready and you want local browser access that survives app rollouts.

### Why this is stable

- It tunnels to Kubernetes **Service ClusterIP** endpoints (not pod port-forward), so pod restarts during deploy do not drop local access.
- It runs a reconnect loop and auto-rebuilds tunnel if SSH disconnects.

### Start (recommended)

From `test-infra` directory:

```bash
./scripts/start-local-exposure.sh 34.41.81.193
```

### Status

```bash
./scripts/status-local-exposure.sh
```

### Stop

```bash
./scripts/stop-local-exposure.sh
```

### Open locally

```bash
# React
http://127.0.0.1:18080

# Node
http://127.0.0.1:18081
```

### Quick verification

```bash
curl -sSI http://127.0.0.1:18080 | head -n 3
curl -sS -i http://127.0.0.1:18081 | head -n 12
```

If Terraform recreates infra, use the new bastion IP in `start-local-exposure.sh`.

### One-time setup (no manual script runs): systemd --user

If you want localhost to stay available automatically (including auto-restart after tunnel drops), use a user service:

```bash
cd ../test-infra
chmod +x scripts/local-exposure-systemd-runner.sh
mkdir -p ~/.config/systemd/user
cp systemd/infra-local-exposure.service ~/.config/systemd/user/

cat > ~/.config/infra-local-exposure.env <<'EOF'
BASTION_IP=<YOUR_BASTION_IP>
REMOTE_USER=ubuntu
REACT_NS=infra-test-react
REACT_SVC=infra-test-react
REACT_LOCAL_PORT=18080
NODE_NS=infra-test-node
NODE_SVC=infra-test-node
NODE_LOCAL_PORT=18081
EOF

systemctl --user daemon-reload
systemctl --user enable --now infra-local-exposure.service
```

Useful commands:

```bash
systemctl --user status infra-local-exposure.service
systemctl --user restart infra-local-exposure.service
journalctl --user -u infra-local-exposure.service -f
```

## Domain troubleshooting (important)

If browser says **"This site can’t be reached"**:

1. Confirm DNS A record points to the actual ingress public IP.
2. Confirm public ports `80/443` are open in cloud firewall to ingress nodes.
3. Confirm ingress controller has healthy pods/endpoints.
4. Confirm ingress host in cluster matches domain.

Useful commands:

```bash
dig +short zubeyr.duckdns.org
dig +short nodejstest.duckdns.org

kubectl -n ingress-nginx get ds,svc,pods,endpoints
kubectl -n infra-test-react get ingress infra-test-react -o yaml
kubectl -n infra-test-node get ingress infra-test-node -o yaml
```

If DNS is correct but still unreachable, issue is usually network path (firewall/NAT/public IP binding), not Argo manifests.

## Add domain host (beside localhost) - end-to-end steps

Use this flow any time you want an app reachable by real domain name (not only `127.0.0.1`).

### 1) Get the current ingress public IP

From `test-infra/terraform/gcp-mvp`:

```bash
terraform output ingress_static_ip
```

### 2) Point DNS A record to that IP

- Create or update A record for your domain/subdomain.
- Example:
  - `react.example.com` -> `<ingress_static_ip>`
  - `node.example.com` -> `<ingress_static_ip>`

### 3) Update ingress manifest host in App-of-Apps

Edit app ingress files:

- `workloads/infra-test-react/ingress.yaml`
- `workloads/infra-test-node/ingress.yaml`

Set:

- `spec.rules[].host` to your domain
- `spec.tls[].hosts[]` to same domain
- keep `cert-manager.io/cluster-issuer: letsencrypt-prod` annotation if you want TLS certs

### 4) Commit and push

```bash
git add workloads/infra-test-react/ingress.yaml workloads/infra-test-node/ingress.yaml
git commit -m "Update ingress hosts for domains"
git push
```

### 5) Wait for Argo sync (or force refresh)

```bash
kubectl -n argocd get app
kubectl annotate app platform-app-of-apps -n argocd argocd.argoproj.io/refresh=hard --overwrite
```

### 6) Verify DNS and ingress

```bash
dig +short react.example.com
dig +short node.example.com

kubectl -n infra-test-react get ingress infra-test-react -o wide
kubectl -n infra-test-node get ingress infra-test-node -o wide
```

### 7) Verify TLS certificate readiness (if HTTPS)

```bash
kubectl -n infra-test-react get certificate,challenge,order
kubectl -n infra-test-node get certificate,challenge,order
```

### 8) Test from browser

- `https://react.example.com`
- `https://node.example.com`

If DNS is correct and apps are still not reachable, check firewall rules for ports `80/443`, ingress controller pod health, and whether ingress static IP is attached to ingress worker path.
