# Helm Chart with Secret Expressions in Values File

This example demonstrates using Harness Secret Expressions in a **Helm `values.yaml` file** that gets templated into Kubernetes Secrets.

## Overview

This pattern shows how to:
- Put secret expressions in the `values.yaml` file
- Reference those values in Helm templates using standard `{{ .Values.* }}` syntax
- Keep your Helm chart **portable** (templates have no vendor-specific code)
- Use secrets stored in **Harness Secret Manager**

## What's Included

```
helm-values-example/
├── README.md                (this file)
├── Chart.yaml               Helm chart metadata
├── values.yaml              ⭐ Values with secret expressions - START HERE
└── templates/
    ├── secret.yaml          ⭐ Template using values - SEE THIS TOO
    ├── deployment.yaml      Sample deployment
    └── service.yaml         Sample service
```

## Secret Expression Locations

### 📍 Expressions in Values: [`values.yaml`](./values.yaml)

The secret expressions are in the values file:

```yaml
# values.yaml - All project-level secrets!
secrets:
  database:
    username: <+secrets.getValue("dbUsername")>
    password: <+secrets.getValue("dbPassword")>
  api:
    key: <+secrets.getValue("apiKey")>
  application:
    secretKey: <+secrets.getValue("appSecret")>
  cache:
    password: <+secrets.getValue("redisPassword")>
```

### 📍 Template Using Values: [`templates/secret.yaml`](./templates/secret.yaml)

The template uses standard Helm syntax (NO vendor lock-in):

```yaml
# templates/secret.yaml
apiVersion: v1
kind: Secret
stringData:
  db-password: {{ .Values.secrets.database.password }}
  api-key: {{ .Values.secrets.api.key }}
```

## Why This Approach?

**✅ Portable Manifests**: Your Helm templates stay vendor-agnostic. You can use the same chart with:
- Harness GitOps (with secret expressions in values)
- Plain Argo CD (with plain values)
- Helm CLI (with any values file)
- Other GitOps tools

**✅ Natural Pattern**: Values files are the standard way to configure Helm charts - this follows that pattern.

**✅ No Duplicate Charts**: You don't need separate Helm charts for different tools or platforms.

## Prerequisites

### 1. Create Secrets in Harness

Create these **project-level** secrets in Harness (backed by **Harness Secret Manager**):

**Project Settings** → **Secrets** → **+ New Secret** → **Text**

Create these 5 secrets:

1. **Identifier**: `apiKey` - API key for services
2. **Identifier**: `dbPassword` - Database password  
3. **Identifier**: `dbUsername` - Database username
4. **Identifier**: `appSecret` - Application secret key
5. **Identifier**: `redisPassword` - Redis/Cache password

> 💡 **Tip**: See [SECRETS-SETUP-GUIDE.md](../SECRETS-SETUP-GUIDE.md) for detailed instructions.

**All secrets should use Harness Secret Manager** as the secret manager.

### 2. GitOps Agent Setup

- Feature flag enabled: `CDS_GITOPS_SECRET_RESOLUTION_ENABLED`
- Harness plugin enabled on GitOps agent (see main [README](../README.md#prerequisites))

## How to Deploy

### Step 1: Review Values

Check [`values.yaml`](./values.yaml) - update secret identifiers if needed:

```yaml
secrets:
  database:
    password: <+secrets.getValue("YOUR_SECRET_ID")>
```

> **Note**: All secrets use project-level scope (no prefix needed!)

### Step 2: Commit to Git

```bash
git add helm-values-example/
git commit -m "Add Helm values secret expression example"
git push
```

### Step 3: Create GitOps Application

In Harness:
1. Go to **Deployments** → **GitOps** → **Applications**
2. Click **+ New Application**
3. Configure:
   - **Name**: `helm-secrets-example`
   - **Agent**: Your agent (with plugin enabled)
   - **Repository**: Your Git repo URL
   - **Path**: `GitOps-Secret-Expressions/helm-values-example`
   - **Cluster**: Your target cluster
   - **Namespace**: `default`
4. Click **Create**

### Step 4: Sync

1. Go to **App Diff** to preview changes
2. Verify secret expressions are resolved (values masked)
3. Click **Sync**

### Step 5: Verify

```bash
# Check Helm release
helm list -n default

# Check secret
kubectl get secret helm-secrets-example -n default

# Check deployment
kubectl get deployment helm-secrets-example -n default
kubectl get pods -l app.kubernetes.io/name=sample-app -n default
```

## How It Works

```
┌──────────────────┐
│  values.yaml     │
│  (in Git)        │
│                  │
│  password:       │
│  <+secrets...>   │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Harness Plugin   │
│                  │
│ Resolves values  │
│ from Vault       │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Helm Template    │
│ Engine           │
│                  │
│ {{.Values...}}   │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Kubernetes       │
│ Secret           │
│                  │
│ password: <val>  │
└──────────────────┘
```

1. **Harness plugin** reads `values.yaml` and resolves `<+secrets.getValue()>` expressions from Harness Secret Manager
2. **Resolved values** are passed to Helm
3. **Helm templates** the manifests using standard `{{ .Values.* }}` syntax
4. **Kubernetes** receives the final Secret with actual values

## Key Points

✅ **Portable**: Templates have zero Harness-specific code  
✅ **Standard Helm**: Uses normal Helm templating patterns  
✅ **Harness Secret Manager**: Secrets stored in Harness Secret Manager  
✅ **Project-Level**: Simple syntax with no prefixes  
✅ **Reusable**: Same chart works with any tool  
✅ **Best Practice**: Follows Helm conventions for values  

## Comparison: This vs Direct Expressions

### ❌ Not Recommended: Expressions in Templates

```yaml
# templates/secret.yaml - DON'T DO THIS
stringData:
  password: <+secrets.getValue("dbPassword")>  # Locks chart to Harness!
```

This makes your chart vendor-specific and non-portable.

### ✅ Recommended: Expressions in Values

```yaml
# values.yaml - DO THIS (project-level)
secrets:
  password: <+secrets.getValue("dbPassword")>

# templates/secret.yaml - Stays portable!
stringData:
  password: {{ .Values.secrets.password }}
```

This keeps templates portable and follows Helm best practices.

## Troubleshooting

### Issue: Template rendering fails

**Check**:
- YAML syntax in values.yaml
- Template references match values structure
- Run `helm template` locally to test (expressions won't resolve but structure will validate)

### Issue: Secrets not resolving

**Check**:
- Harness plugin is enabled on agent
- Secrets exist in Harness with correct identifiers
- Secrets are backed by Harness Secret Manager
- Feature flag is enabled

### Issue: "undefined value" error from Helm

**Cause**: Template references a value that doesn't exist

**Solution**:
```yaml
# values.yaml must have the structure referenced in templates
secrets:
  database:
    password: <+secrets.getValue("dbPassword")>  # This must exist

# templates/secret.yaml
# This path must match values structure
{{ .Values.secrets.database.password }}
```

## Next Steps

- Try the [simple example](../simple-example/) for expressions directly in manifests
- Check the [shared manifests example](../shared-manifests-example/) for using with CD Pipelines
- Learn about [Helm value file precedence](https://helm.sh/docs/chart_template_guide/values_files/)

## Clean Up

```bash
# Via Harness UI: Applications → helm-secrets-example → Delete

# Or using kubectl:
kubectl delete all -l app.kubernetes.io/instance=helm-secrets-example -n default
kubectl delete secret helm-secrets-example -n default
```

---

**Related**: See main [README](../README.md) for complete documentation and troubleshooting.
