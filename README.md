# Argo CD App-of-Apps (Infra Test)

This repository is the GitOps source used by Argo CD App-of-Apps.

Current apps:
- `infra-test-react` → host: `zubeyr.duckdns.org`
- `infra-test-node` → host: `nodejstest.duckdns.org`

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
