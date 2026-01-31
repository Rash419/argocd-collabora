# ArgoCD Pipeline for Collabora + Cool-Controller

This repository contains the ArgoCD configuration to automatically deploy the Collabora Online and Cool-Controller using the umbrella helm chart.

## Overview

- **Chart Source:** Private GitLab OCI Registry (`registry.gitlab.collabora.com`)
- **Chart:** `productivity/cool-controller-registry/helm-charts/collabora-online-umbrella`
- **Version:** Latest (automatically fetches new chart versions)
- **Deployment Strategy:** Automatic sync with prune and self-heal enabled
- **Target Namespace:** `collabora`
- **GitOps Approach:** ApplicationSet with multi-source configuration (Git + Helm registry)

## Prerequisites

1. **ArgoCD installed** in your Kubernetes cluster
2. **Ingress Controller** (nginx) installed
3. **Image Pull Secret** (`controller-regcred`) for pulling COOL docker images

## Files

### 1. `applicationset.yaml`

**ApplicationSet** manifest that automatically generates the ArgoCD Application. Key features:

- **Git Generator:** Watches this repository for changes to `values.yaml`
- **Multi-Source:** Combines Git repo (for values) + Helm registry (for chart)
- **Auto-Update:** Fetches latest chart version automatically (`targetRevision: "*"`)
- **GitOps Native:** Changes to `values.yaml` automatically trigger re-deployment

### 2. `values.yaml`

Helm values configuration including:

- Collabora Online settings (replicas, ingress, resources)
- Cool-Controller settings (replicas, ingress, document migration)
- Autoscaling configuration
- Resource limits and requests

### 3. `secret.yaml`

GitLab OCI registry authentication secret for ArgoCD to pull the private helm chart.

## Deployment Steps

### Step 0: Configure Registry Credentials

Before applying secrets, update `secret.yaml` with your GitLab credentials:

1. Open `secret.yaml`
2. Replace `<user_name>` with your GitLab username
3. Replace `<access_token>` with your GitLab access token
4. Save the file

**Note:** Never commit actual credentials to public repositories!

### Step 1: Create the GitLab Registry Secret

Apply the configured secret to allow ArgoCD to access the private OCI registry:

```bash
kubectl apply -f secret.yaml
```

### Step 2: Create Image Pull Secret

Create the `controller-regcred` secret in the `collabora` namespace for pulling COOL docker images:

```bash
kubectl create namespace collabora
kubectl create secret docker-registry controller-regcred \
  --docker-server=registry.gitlab.collabora.com \
  --docker-username=<your-gitlab-username> \
  --docker-password=<your-gitlab-token> \
  --namespace=collabora
```

### Step 3: Create TLS Secret (if needed)

Create the TLS secret for the ingress:

```bash
kubectl create secret tls tls-secret-name \
  --cert=path/to/cert.pem \
  --key=path/to/key.pem \
  --namespace=collabora
```

### Step 4: Update Configuration Values

Before deploying, update the following placeholders in `values.yaml`:

1. **Admin Console Credentials:**
   - `collabora.username`: `<your-admin-console-username>`
   - `collabora.password`: `<your-admin-console-password>`

2. **Domain Names:**
   - Update `test.collabora.online` to your actual domain
   - Update `test.integrator.com` to your integrator domain

3. **TLS Secret:**
   - Update `tls-secret-name` to your actual TLS secret name

4. Edit the configuration as per your wish

### Step 5: Apply the ApplicationSet

```bash
kubectl apply -f applicationset.yaml
```

This creates an ApplicationSet that:

- Watches this Git repository for changes to `values.yaml`
- Automatically generates the `collabora` Application
- Fetches the latest chart version from the Helm registry

### Step 6: Verify Deployment

Check the ApplicationSet and generated Application:

```bash
# View the ApplicationSet
kubectl get applicationset -n argocd collabora-appset

# View the generated Application
kubectl get application -n argocd collabora

# View pods
kubectl get pods -n collabora
```

Or via ArgoCD UI/CLI:

```bash
argocd app get collabora
```

## Troubleshooting

### Check Application Status

```bash
argocd app get collabora
```

### View Logs

```bash
# Collabora pods
kubectl logs -n collabora -l app.kubernetes.io/name=collabora-online

# Controller pods
kubectl logs -n collabora -l app.kubernetes.io/name=cool-controller
```

### Sync Issues

If the application is out of sync:

```bash
argocd app sync collabora
```

### Registry Authentication Issues

Verify the secret is correctly created:

```bash
kubectl get secret gitlab-oci-registry -n argocd -o yaml
```

## Updates

### Updating Configuration (values.yaml)

1. Modify `values.yaml` with new configuration
2. Commit and push to the repository
3. **ApplicationSet automatically detects the change** and triggers re-deployment

### Updating Chart Version

The ApplicationSet is configured with `targetRevision: "*"`, which means:

- **ArgoCD automatically fetches the latest chart version** from the Helm registry
- No manual intervention required
- To pin to a specific version, update `targetRevision` in `applicationset.yaml`

### How It Works

The ApplicationSet uses a **multi-source configuration**:

- **Source 1:** Helm chart from OCI registry (auto-updates to latest version)
- **Source 2:** Git repository (provides `values.yaml`)

ArgoCD polls both sources and automatically syncs when either changes.

## Creating Your Own Repository

Want to set up a similar deployment? Here's the structure:

### Repository Structure

```
your-repo/
├── applicationset.yaml    # ArgoCD ApplicationSet (watches this repo)
├── values.yaml           # Helm values (app configuration)
└── secret.yaml           # OCI registry credentials (gitignored in production)
```

### Key Configuration Points

1. **Change the Git repo URL** in `applicationset.yaml`:
   - Line 8: Update `repoURL` to your Git repository

2. **Change the Helm registry** (if different):
   - Line 19: Update `repoURL` to your chart registry
   - Line 20: Update `chart` path to your chart
   - Line 13 in `secret.yaml`: Update registry URL

3. **Update namespace** (optional):
   - Line 31: Change `namespace: collabora` to your target namespace
   - Update in `values.yaml` references

4. **Customize values.yaml**:
   - Domain names
   - Resource limits
   - Ingress configuration
   - Credentials

### Multi-Source Pattern Explained

This setup uses ArgoCD's multi-source feature to:

- Track Git repository for configuration changes (values.yaml)
- Track Helm registry for chart updates
- Both sources independently trigger deployments

Perfect for teams that want:

- GitOps for configuration (values in Git)
- Automatic chart updates (latest Helm versions)
- Decoupled versioning (config vs chart versions)

## Security Notes

- The `secret.yaml` contains sensitive credentials. Do not commit it to public repositories.
- Consider using external secret management tools (e.g., External Secrets Operator, Vault) for production.
