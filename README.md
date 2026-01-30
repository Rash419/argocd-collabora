# ArgoCD Pipeline for Collabora + Cool-Controller

This repository contains the ArgoCD configuration to automatically deploy the Collabora Online and Cool-Controller using the umbrella helm chart.

## Overview

- **Chart Source:** Private GitLab OCI Registry (`registry.gitlab.collabora.com`)
- **Chart:** `productivity/cool-controller-registry/helm-charts/collabora-online-umbrella`
- **Version:** 1.1.6
- **Deployment Strategy:** Automatic sync with prune and self-heal enabled
- **Target Namespace:** `collabora`

## Prerequisites

1. **ArgoCD installed** in your Kubernetes cluster
2. **Ingress Controller** (nginx) installed
3. **TLS Secret** created for `test.collabora.online`
4. **Image Pull Secret** (`controller-regcred`) for pulling COOL docker images

## Files

### 1. `collabora.yaml`

ArgoCD Application manifest that defines:

- Helm chart source from OCI registry
- Automatic sync policy with retry logic
- Namespace creation enabled
- Values file reference

### 2. `values.yaml`

Helm values configuration including:

- Collabora Online settings (replicas, ingress, resources)
- Cool-Controller settings (replicas, ingress, document migration)
- Autoscaling configuration
- Resource limits and requests

### 3. `secret.yaml`

GitLab OCI registry authentication secret for ArgoCD to pull the private helm chart.

## Deployment Steps

### Step 1: Create the GitLab Registry Secret

Apply the secret to allow ArgoCD to access the private OCI registry:

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

### Step 3: Create TLS Secret

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

### Step 5: Apply the ArgoCD Application

```bash
kubectl apply -f collabora.yaml
```

### Step 6: Verify Deployment

Check the application status in ArgoCD UI or via CLI:

```bash
argocd app get collabora
```

Or using kubectl:

```bash
kubectl get applications -n argocd collabora
kubectl get pods -n collabora
```

## Configuration Details

### Collabora Online

- **Replicas:** 2 (with autoscaling enabled)
- **Resources:**
  - CPU: 4000m request / 8000m limit
  - Memory: 6000Mi request / 8000Mi limit
- **Autoscaling:**
  - Target CPU: 60%
  - Target Memory: 80%
- **Ingress:** nginx with RouteToken-based routing

### Cool-Controller

- **Replicas:** 2 (with leader election for HA)
- **Features:**
  - Document migration enabled
  - Hashmap parallelization enabled
  - Stats interval: 2000ms
- **Ingress:** nginx with `/controller` path

## Accessing the Application

Once deployed, you can access:

- **Collabora Online:** <https://test.collabora.online>
- **Cool-Controller:** <https://test.collabora.online/controller>
- **Admin Console:** <https://test.collabora.online/browser/dist/admin/admin.html>

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

To update the deployment:

1. Modify `values.yaml` with new configuration
2. Commit and push to the repository
3. ArgoCD will automatically sync the changes

To upgrade the chart version, update `targetRevision` in `collabora.yaml`.

## Security Notes

- The `secret.yaml` contains sensitive credentials. Do not commit it to public repositories.
- Consider using external secret management tools (e.g., External Secrets Operator, Vault) for production.
- Rotate GitLab tokens regularly.

## Support

For issues related to:

- **ArgoCD:** <https://argo-cd.readthedocs.io/>
- **Collabora Online:** <https://collaboraonline.github.io/>
- **Cool-Controller:** Contact Collabora support
