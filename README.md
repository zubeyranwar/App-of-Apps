# Argo CD App-of-Apps (Test Setup)

This repository directory is the GitOps source for a **test application** (`infra-test-react`) using an App-of-Apps pattern.

Goal:
- Push image from GitHub Actions to GHCR
- Argo CD Image Updater detects GHCR update
- Image Updater writes new image reference back to Git
- Argo CD syncs and deploys the new version automatically

## Structure

- `bootstrap/`:
  - `project.yaml`: Argo CD project definition
  - `root-application.yaml`: root app that manages all child apps in `apps/`
- `apps/infra-test-react/`:
  - child Argo CD `Application` with image-updater annotations
- `workloads/infra-test-react/`:
  - Kubernetes manifests (Kustomize) for deployment/service/ingress

## Current configured values

- GitOps repo: `https://github.com/zubeyranwar/App-of-Apps.git`
- App image: `ghcr.io/zubeyranwar/infra-test-react:main`

## Required one-time edit

1. In `workloads/infra-test-react/ingress.yaml`:
	- `react-test.example.com` (set your real DNS host)

## Deploy App-of-Apps

```bash
kubectl apply -f bootstrap/project.yaml
kubectl apply -f bootstrap/root-application.yaml
kubectl apply -f bootstrap/image-updater.yaml
```

## Verify image auto-update flow

1. Push to `infra-test-react` repo `main` branch.
2. GitHub Action builds and pushes `ghcr.io/<user>/infra-test-react:main`.
3. Argo CD Image Updater updates `workloads/infra-test-react/kustomization.yaml` in Git.
4. Argo CD syncs and rollout happens in namespace `infra-test-react`.

## How to test without a real domain

You do **not** need DNS first to verify that GitOps works.

Use this order:

1. Push the React app repo (`infra-test-react`) so GitHub Actions publishes the image to GHCR.
2. Push the GitOps repo (`App-of-Apps`) so Argo CD can read the manifests.
3. Apply the bootstrap manifests.
4. Confirm Argo CD creates the app and the `infra-test-react` pods become `Running`.
5. Test the app with `kubectl port-forward` to the service in namespace `infra-test-react`.

Example checks:

- `kubectl get applications -n argocd`
- `kubectl get pods -n infra-test-react`
- `kubectl get svc -n infra-test-react`
- `kubectl port-forward svc/infra-test-react 8080:80 -n infra-test-react`

Then open `http://localhost:8080`.

If this works, your deployment path is correct even without ingress DNS.

## What “domain” means here

The value in `workloads/infra-test-react/ingress.yaml` is only the public hostname for ingress, for example `app.infratest.vps.thec1oud.uk`.

It is **not required** for the first GitOps test.

You only need it later if you want to reach the app from the internet through ingress and TLS.

## Notes

- This setup assumes Argo CD and Argo CD Image Updater are already installed.
- Image Updater uses `digest` strategy on tag `main`.
- Workload manifests are intentionally minimal for infra/GitOps validation.
