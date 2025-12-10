# Simple Kubernetes Secret with Harness Expressions

This example demonstrates the simplest way to use Harness Secret Expressions: **directly in a Kubernetes Secret manifest**.

## Overview

This is a straightforward example where secret expressions are placed directly in the `stringData` field of a Kubernetes Secret resource. The secrets are stored in **Harness Secret Manager** and managed through Harness.

## What's Included

```
simple-example/
├── README.md           (this file)
├── secret.yaml         ⭐ Secret with expressions - START HERE
├── deployment.yaml     Sample app using the secrets
└── service.yaml        Sample service
```

## Secret Expression Location

**📍 See the expressions here**: [`secret.yaml`](./secret.yaml)

The secret expressions are in the `stringData` field:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
stringData:
  # All project-level secrets - simple syntax!
  api-key: <+secrets.getValue("apiKey")>
  db-password: <+secrets.getValue("dbPassword")>
  db-username: <+secrets.getValue("dbUsername")>
  app-secret: <+secrets.getValue("appSecret")>
  redis-password: <+secrets.getValue("redisPassword")>
```

## Prerequisites

### 1. Create Secrets in Harness

Create these **project-level** secrets in Harness (backed by **Harness Secret Manager**):

**Project Settings** → **Secrets** → **+ New Secret** → **Text**

Create these 5 secrets:

1. **API Key**
   - **Identifier**: `apiKey`
   - **Secret Manager**: Harness Secret Manager
   - **Value**: Your API key

2. **Database Password**
   - **Identifier**: `dbPassword`
   - **Secret Manager**: Harness Secret Manager
   - **Value**: Your database password

3. **Database Username**
   - **Identifier**: `dbUsername`
   - **Secret Manager**: Harness Secret Manager
   - **Value**: Your database username

4. **Application Secret**
   - **Identifier**: `appSecret`
   - **Secret Manager**: Harness Secret Manager
   - **Value**: Your application secret key

5. **Redis Password**
   - **Identifier**: `redisPassword`
   - **Secret Manager**: Harness Secret Manager
   - **Value**: Your Redis password

> 💡 **Tip**: See [SECRETS-SETUP-GUIDE.md](../SECRETS-SETUP-GUIDE.md) for detailed step-by-step instructions.

### 2. Enable GitOps Agent Plugin

- **New agent**: Check "Enable Argo CD Harness Plugin" during installation
- **Existing agent**: Run the patch script (see main [README](../README.md#prerequisites))

### 3. Enable Feature Flag

Contact Harness Support to enable: `CDS_GITOPS_SECRET_RESOLUTION_ENABLED`

## How to Deploy

### Step 1: Update Secret Identifiers (if needed)

If your Harness secrets have different identifiers, update [`secret.yaml`](./secret.yaml):

```yaml
stringData:
  api-key: <+secrets.getValue("YOUR_SECRET_ID")>
```

> **Note**: All secrets use project-level scope (no prefix needed!)

### Step 2: Commit to Git

```bash
git add simple-example/
git commit -m "Add simple secret expression example"
git push
```

### Step 3: Create GitOps Application

In Harness:
1. Go to **Deployments** → **GitOps** → **Applications**
2. Click **+ New Application**
3. Configure:
   - **Name**: `simple-secrets-example`
   - **Agent**: Your agent (with Harness plugin enabled)
   - **Repository**: Your Git repo URL
   - **Path**: `GitOps-Secret-Expressions/simple-example`
   - **Cluster**: Your target cluster
   - **Namespace**: `default`
4. Click **Create**

### Step 4: Review and Sync

1. Go to **App Diff** tab
2. Verify:
   - ✅ Secret expressions are resolved
   - ✅ Values are masked in the UI (for security)
   - ✅ No errors
3. Click **Sync**

### Step 5: Verify

```bash
# Check secret exists
kubectl get secret app-secrets -n default

# Verify deployment is running
kubectl get deployment sample-app -n default
kubectl get pods -l app=sample-app -n default

# Check app logs
kubectl logs -l app=sample-app -n default
```

## How It Works

```
┌─────────────────┐
│  secret.yaml    │
│  (in Git)       │
│                 │
│  vault-secret:  │
│  <+secrets...>  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Harness Plugin  │
│ (in Argo CD)    │
│                 │
│ Resolves expr.  │
│ from Vault      │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Kubernetes      │
│ Secret Resource │
│                 │
│ vault-secret:   │
│ <actual-value>  │
└─────────────────┘
```

1. **You commit** `secret.yaml` with expressions to Git
2. **Argo CD renders** the manifest
3. **Harness plugin** intercepts and resolves `<+secrets.getValue()>` expressions
4. **Harness fetches** the actual values from Harness Secret Manager
5. **Kubernetes creates** the Secret with resolved values
6. **Values are masked** in Harness and Argo CD UIs

## Key Points

✅ **Expressions in Secret Resource**: Only works in `kind: Secret` resources (security feature)  
✅ **Harness Secret Manager**: Secrets are stored in Harness Secret Manager  
✅ **Project-Level Only**: Simple syntax with no prefixes needed  
✅ **Masked in UI**: Actual values never shown in Harness or Argo CD interfaces  
✅ **No Git Secrets**: Sensitive data never committed to Git  

## Troubleshooting

### Issue: Expressions not resolving

**Check**:
1. Harness plugin enabled on GitOps agent
2. Feature flag `CDS_GITOPS_SECRET_RESOLUTION_ENABLED` is enabled
3. Secrets exist in Harness with exact identifiers
4. Secrets are backed by Harness Secret Manager
5. You have RBAC permissions to access the secrets

### Issue: "Secret not found" error

**Solution**:
- Verify secret identifier matches exactly (case-sensitive)
- Confirm secret exists in **Project Settings** → **Secrets**
- Ensure secret is backed by Harness Secret Manager

### Issue: Values visible in plain text

**Expected**: Values should be masked as `***` in Harness UI

**If not**:
- Ensure you're viewing through Harness/Argo CD UI (not direct kubectl)
- Kubernetes base64-encodes values - use `kubectl get secret -o yaml` to see encoded

## Next Steps

- Try the [Helm example](../helm-values-example/) to see expressions in values files
- Check the [shared manifests example](../shared-manifests-example/) to use same manifests with CD Pipelines
- Learn about [updating secrets](../README.md#updating-secrets)

## Clean Up

```bash
# Delete via Harness UI: Applications → simple-secrets-example → Delete

# Or using kubectl:
kubectl delete -f simple-example/ -n default
```

---

**Related**: See main [README](../README.md) for complete documentation and how secret expressions work.
