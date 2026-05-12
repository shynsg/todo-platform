# Git, Image, And Argo CD Production Flow

## Should The Server Pull Git Source?

Production Kubernetes should not run app source code by `git pull` on the server.

Recommended flow:

```text
Git repo source code
-> CI builds Docker image
-> CI pushes image to registry
-> GitOps repo stores Kubernetes manifests with image tag
-> Argo CD syncs manifests into cluster
-> Kubernetes pulls image from registry
```

Git stores:

```text
source code
Dockerfile
Kubernetes manifests / Helm / Kustomize
```

Registry stores:

```text
built application image
```

Kubernetes runs:

```text
container image
```

## Repos For This Capstone

Use 3 repos:

```text
todo-frontend.git
todo-backend.git
todo-platform.git
```

`todo-frontend.git`:

```text
React source
Dockerfile
GitHub Actions build/push image
```

`todo-backend.git`:

```text
Express source
Dockerfile
migration script
GitHub Actions build/push image
```

`todo-platform.git`:

```text
Kustomize manifests
Argo CD Application
monitoring manifests
```

## Image Names

Example GHCR images:

```text
ghcr.io/YOUR_ORG/todo-frontend:<git-sha>
ghcr.io/YOUR_ORG/todo-backend:<git-sha>
```

Update Kustomize image tags in platform repo:

```yaml
images:
  - name: ghcr.io/YOUR_ORG/todo-backend
    newTag: <git-sha>
  - name: ghcr.io/YOUR_ORG/todo-frontend
    newTag: <git-sha>
```

Then Argo CD syncs.

## What Runs On The Server?

Server runs:

```text
K3s
Argo CD
app workloads
Traefik ingress
monitoring
```

Server does not need:

```text
git pull app source
npm install app source
docker build app source
```

Those belong in CI.

## No Domain Yet

For practice without a domain:

```text
http://SERVER_IP/
http://SERVER_IP/api/health
```

Use an Ingress without `host`.

For production:

```text
buy/use a domain
point DNS A record to server IP
add cert-manager
enable HTTPS
```

TLS with only raw IP is possible but awkward and not the normal web production setup.
