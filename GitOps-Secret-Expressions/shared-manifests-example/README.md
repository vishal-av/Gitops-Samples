# Shared Manifests: GitOps AND CD Pipeline

This example demonstrates a crucial point: **You do NOT need separate manifests for GitOps vs CD Pipelines!**

## Overview

The same Git repository and manifests can be used by:
- ✅ **GitOps Application** (Argo CD syncing from Git)
- ✅ **CD Pipeline** (Traditional Harness pipeline deployment)

Secret expressions work the same way in both scenarios.

## What's Included

```
shared-manifests-example/
├── README.md               (this file)
├── manifests/             
│   ├── secret.yaml         Secret with Harness expressions
│   ├── deployment.yaml     Deployment using secrets
│   └── service.yaml        Service definition
├── gitops-application.yaml   GitOps Application definition
└── cd-pipeline.yaml         CD Pipeline definition (for reference)
```

## The Key Insight

```
┌─────────────────────────────────┐
│  Git Repository                 │
│  └── manifests/                 │
│      ├── secret.yaml            │
│      │   <+secrets.getValue()>  │
│      ├── deployment.yaml        │
│      └── service.yaml           │
└────────────┬────────────────────┘
             │
         ┌───┴───┐
         │       │
         ▼       ▼
    ┌─────┐   ┌─────┐
    │GitOps│   │ CD  │
    │ App │   │ Pipe│
    └──┬──┘   └──┬──┘
       │         │
       └────┬────┘
            │
            ▼
    ┌──────────────┐
    │  Kubernetes  │
    │   Cluster    │
    └──────────────┘
```

**Both paths work with the same manifests!**

## Use Case 1: GitOps Application

Deploy using Harness GitOps (Argo CD):

### Setup GitOps Application

1. In Harness UI: **Deployments** → **GitOps** → **Applications**
2. Click **+ New Application**
3. Configure:
   - **Name**: `shared-manifests-app`
   - **Agent**: Your GitOps agent
   - **Repository**: Your Git repo URL
   - **Path**: `GitOps-Secret-Expressions/shared-manifests-example/manifests`
   - **Cluster**: Your cluster
   - **Namespace**: `default`
4. Click **Create** and **Sync**

### How It Works

1. Argo CD monitors Git repository
2. On changes, Harness plugin resolves secret expressions
3. Kubernetes resources are created/updated
4. Application syncs automatically (if auto-sync enabled)

## Use Case 2: CD Pipeline

Deploy using traditional Harness CD Pipeline:

### Setup CD Pipeline

1. Create a **Harness Service**:
   - Type: Kubernetes
   - Manifests Source: Git Repository
   - Repository: Same Git repo
   - Path: `GitOps-Secret-Expressions/shared-manifests-example/manifests`

2. Create a **Pipeline**:
   - Add Deploy stage
   - Select the service created above
   - Select your environment
   - Choose Rolling deployment

3. Run the pipeline

### How It Works

1. Pipeline execution starts
2. Harness fetches manifests from Git
3. Secret expressions are resolved
4. kubectl apply executes manifests
5. Resources deployed to cluster

## The Manifests

These are the **same manifests** used by both GitOps and CD Pipeline:

### `manifests/secret.yaml`

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: shared-app-secrets
stringData:
  # All project-level secrets
  api-key: <+secrets.getValue("apiKey")>
  db-password: <+secrets.getValue("dbPassword")>
  db-username: <+secrets.getValue("dbUsername")>
  app-secret: <+secrets.getValue("appSecret")>
```

### `manifests/deployment.yaml`

Standard Kubernetes Deployment referencing the Secret.

### `manifests/service.yaml`

Standard Kubernetes Service.

## Prerequisites

Same as other examples:

1. **Secrets in Harness** (backed by HashiCorp Vault):
   - Project-level: `apiKey`, `dbPassword`, `dbUsername`, `appSecret`

2. **For GitOps**: Agent with Harness plugin enabled
3. **For CD Pipeline**: Kubernetes connector configured
4. **Feature Flag**: `CDS_GITOPS_SECRET_RESOLUTION_ENABLED`

> 💡 See [SECRETS-SETUP-GUIDE.md](../SECRETS-SETUP-GUIDE.md) for creating the secrets.

## Deploy Both Ways

You can even deploy the same application using both methods simultaneously:

1. **Production**: Use GitOps for automated sync
2. **Development**: Use CD Pipeline for manual deployments

Both will work with the same Git repository and manifests!

## Key Benefits

✅ **No Duplication**: Single source of truth in Git  
✅ **Flexibility**: Choose deployment method per environment  
✅ **Consistency**: Same manifests = same behavior  
✅ **Easy Migration**: Move from CD Pipeline to GitOps (or vice versa) without changing manifests  

## Testing

### Deploy via GitOps:
```bash
# Application will sync automatically or manually trigger sync
kubectl get all -l app=shared-manifests -n default
```

### Deploy via CD Pipeline:
```bash
# Run pipeline in Harness UI
# Or trigger via API
curl -X POST https://app.harness.io/pipeline/api/webhook/custom/...
```

### Verify:
```bash
kubectl get secret shared-app-secrets -n default
kubectl get deployment shared-manifests -n default
kubectl logs -l app=shared-manifests -n default
```

## When to Use Which?

### Use GitOps When:
- You want automatic sync from Git
- Multiple teams need to deploy
- You prefer declarative state management
- You want drift detection and reconciliation

### Use CD Pipeline When:
- You need complex deployment logic
- You want manual approval gates
- You need pre/post-deployment hooks
- You want integration with other CD stages (build, test, etc.)

### Use Both When:
- Different environments have different needs
- You're migrating from one approach to the other
- You want flexibility in deployment methods

## Troubleshooting

### GitOps vs Pipeline Conflict

**Issue**: Both GitOps and Pipeline deploying to same namespace

**Solution**: Choose one method per environment:
- Prod: GitOps
- Dev: CD Pipeline
Or use different namespaces.

### Secrets Not Resolving in Pipeline

**Check**:
- Service manifest source is correctly configured
- Secrets exist in Harness
- Pipeline has access to secrets (RBAC)

### Secrets Not Resolving in GitOps

**Check**:
- GitOps agent has Harness plugin enabled
- Feature flag is enabled
- Application is syncing correctly

## Clean Up

```bash
# If deployed via GitOps:
# Delete application in Harness GitOps UI

# If deployed via Pipeline:
# Run rollback pipeline or manually delete

# Or delete resources directly:
kubectl delete -f manifests/ -n default
```

## Next Steps

- Review [simple example](../simple-example/) for basic secret expressions
- Check [Helm example](../helm-values-example/) for Helm-based deployments
- Learn about [GitOps vs CD decisions](https://developer.harness.io/docs/continuous-delivery/gitops/

)

---

**Key Takeaway**: One set of manifests, multiple deployment methods. No need for duplication! 🎉

