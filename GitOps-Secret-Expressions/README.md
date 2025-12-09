# Harness Secret Expressions in GitOps Applications

This guide demonstrates how to use Harness Secret Expressions in GitOps Applications manifests. This feature allows you to manage secrets centrally in Harness and reference them securely in your Kubernetes manifests without storing sensitive data in Git.

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Supported Secret Managers](#supported-secret-managers)
- [How It Works](#how-it-works)
- [Secret Expression Syntax](#secret-expression-syntax)
- [Where Secret Expressions Work](#where-secret-expressions-work)
- [Example Applications](#example-applications)
- [Setup Instructions](#setup-instructions)
- [Important Notes and Gotchas](#important-notes-and-gotchas)
- [Troubleshooting](#troubleshooting)

---

## Overview

The Harness Secret Expressions feature in GitOps provides a secure way to manage secrets across all your applications without:
- Encrypting secrets in manifest files stored in Git
- Manually installing and managing additional plugins (like Vault plugin) on GitOps clusters
- Duplicating secret definitions across multiple applications

Instead, you can:
- Create secrets once in Harness (at Account, Org, or Project level)
- Reference them across any application and manifest using expressions
- Reuse secrets already configured for Harness Pipelines
- Centrally manage and rotate secrets from one location

---

## Prerequisites

Before using this feature, ensure you have:

1. **Feature Flag Enabled**: `CDS_GITOPS_SECRET_RESOLUTION_ENABLED`
   - Contact Harness Support to enable this feature flag

2. **GitOps Agent Configuration**:
   
   **For New Installations:**
   - Check the **"Enable Argo CD Harness Plugin (Required for Harness expression resolution)"** checkbox during agent installation
   
   **For Existing Installations:**
   - Run the following patch script on your cluster to enable the Harness plugin:
   
   ```bash
   # Set your GitOps agent identifier
   AGENT_IDENTIFIER="your-agent-identifier"
   
   # Patch the argocd-repo-server deployment to enable the plugin
   kubectl patch deployment argocd-repo-server -n argocd --type='json' \
     -p='[{
       "op": "add",
       "path": "/spec/template/spec/initContainers/-",
       "value": {
         "name": "harness-secret-plugin-init",
         "image": "harness/gitops-agent:latest",
         "command": ["/bin/sh", "-c"],
         "args": ["cp /usr/local/bin/argocd-vault-plugin /custom-tools/"],
         "volumeMounts": [{
           "name": "custom-tools",
           "mountPath": "/custom-tools"
         }]
       }
     }]'
   
   # Add the custom-tools volume if not already present
   kubectl patch deployment argocd-repo-server -n argocd --type='json' \
     -p='[{
       "op": "add",
       "path": "/spec/template/spec/volumes/-",
       "value": {
         "name": "custom-tools",
         "emptyDir": {}
       }
     }]'
   
   # Restart the argocd-repo-server to apply changes
   kubectl rollout restart deployment argocd-repo-server -n argocd
   ```
   
   > **Note**: Replace `your-agent-identifier` with your actual GitOps agent identifier and adjust the namespace if your Argo CD installation uses a different namespace.

3. **Secrets Created in Harness with HashiCorp Vault**:
   - Create secrets in Harness at Account, Org, or Project level
   - Configure HashiCorp Vault or Harness Secret Manager as the backend
   - **For Vault**: Ensure Vault connector is properly configured in Harness

4. **Appropriate RBAC Permissions**:
   - Permissions to create and manage secrets in Harness
   - Permissions to create/manage GitOps applications

---

## Supported Secret Managers

**Currently Supported:**
- **HashiCorp Vault**
- **Harness Secret Manager (Built-in)**

**Not Currently Supported:**
- AWS Secrets Manager
- Azure Key Vault
- GCP Secret Manager
- Other third-party secret managers

---

## How It Works

1. **Create Secret in Harness**: Define your secret once in Harness at the appropriate scope (Account/Org/Project)

2. **Reference in Manifest**: Use the secret expression syntax in your Kubernetes Secret manifests

3. **Manifest Rendering**: When Argo CD renders the manifest, the Harness plugin resolves and decrypts the secret expressions

4. **Sync to Cluster**: After reviewing the diff, sync the application to push the resolved secret values to your cluster

5. **Security**: Kubernetes automatically base64-encodes the values (when using `stringData`) and masks them in the UI

---

## Secret Expression Syntax

The syntax varies based on the scope where the secret is created:

| Scope | Expression Syntax | Example |
|-------|-------------------|---------|
| **Account Level** | `<+secrets.getValue("account.SECRET_ID")>` | `<+secrets.getValue("account.vaultSecret")>` |
| **Organization Level** | `<+secrets.getValue("org.SECRET_ID")>` | `<+secrets.getValue("org.apiKey")>` |
| **Project Level** | `<+secrets.getValue("SECRET_ID")>` | `<+secrets.getValue("dbPassword")>` |

> **Important**: Replace `SECRET_ID` with the actual identifier of your secret as configured in Harness.

---

## Where Secret Expressions Work

### ✅ **SUPPORTED LOCATIONS**

Secret expressions are resolved in the following locations:

#### 1. **Kubernetes Secret Manifests (stringData or data fields)**
Secret expressions **ONLY** work in Kubernetes resources of `kind: Secret`. This is a security measure - Kubernetes and Argo CD automatically obfuscate secret values in UI, logs, and diffs.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
stringData:
  api-key: <+secrets.getValue("account.apiKey")>
  password: <+secrets.getValue("dbPassword")>
```

#### 2. **Helm Chart values.yaml Files**
You can use secret expressions in your `values.yaml` files, which then get templated into Secret resources:

```yaml
# values.yaml
database:
  username: <+secrets.getValue("org.dbUsername")>
  password: <+secrets.getValue("org.dbPassword")>
```

```yaml
# templates/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
stringData:
  username: {{ .Values.database.username }}
  password: {{ .Values.database.password }}
```

#### 3. **ApplicationSet config.json Files**
For ApplicationSet patterns, you can use expressions in your config files:

```json
{
  "apiKey": "<+secrets.getValue(\"account.apiKey\")>",
  "dbPassword": "<+secrets.getValue(\"org.dbPassword\")>"
}
```

#### 4. **Plain Kubernetes Manifests**
Direct usage in plain Kubernetes Secret manifests without Helm:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: redis-credentials
stringData:
  redis-password: <+secrets.getValue("account.redisPassword")>
```

### ❌ **NOT SUPPORTED LOCATIONS**

Secret expressions will **NOT** be resolved in:

- ❌ **Deployments** (`kind: Deployment`)
- ❌ **ConfigMaps** (`kind: ConfigMap`)
- ❌ **Services** (`kind: Service`)
- ❌ **Ingress** (`kind: Ingress`)
- ❌ **Any non-Secret Kubernetes resource**

**Why?** This is a security feature. If secrets were resolved in non-Secret resources, the actual secret values could be visible in:
- Argo CD UI diffs
- Kubernetes describe commands
- Application logs
- Manifest views

By restricting to Secret resources only, Kubernetes and Argo CD automatically handle obfuscation.

---

## Sample Applications

This repository includes **complete, working sample applications** that demonstrate Harness Secret Expressions in action. These samples show you exactly where to put the secret expressions and how they work in real scenarios.

### 📁 Sample 1: Simple Kubernetes Secret Manifest ([`/simple-example`](./simple-example/))

**The simplest approach** - secret expressions directly in a Kubernetes Secret manifest.

**Where are the secret expressions?**  
→ **[`simple-example/secret.yaml`](./simple-example/secret.yaml)** - Look in the `stringData` field

```yaml
# simple-example/secret.yaml
apiVersion: v1
kind: Secret
stringData:
  vault-secret: <+secrets.getValue("account.vaultSecret")>
  api-key: <+secrets.getValue("org.apiKey")>
  db-password: <+secrets.getValue("dbPassword")>
```

**Secrets are stored in**: HashiCorp Vault (configured in Harness)

**Perfect for**: Teams using plain Kubernetes manifests

[→ View complete simple example with setup instructions](./simple-example/)

---

### 📁 Sample 2: Helm Chart with Values File ([`/helm-values-example`](./helm-values-example/))

**The Helm approach** - secret expressions in `values.yaml`, templates stay portable.

**Where are the secret expressions?**  
→ **[`helm-values-example/values.yaml`](./helm-values-example/values.yaml)** - Expressions are in the values file  
→ **[`helm-values-example/templates/secret.yaml`](./helm-values-example/templates/secret.yaml)** - Template uses standard `{{ .Values.* }}` syntax (no Harness-specific code!)

```yaml
# values.yaml - Expressions here
secrets:
  database:
    password: <+secrets.getValue("org.dbPassword")>
  api:
    key: <+secrets.getValue("apiKey")>
```

```yaml
# templates/secret.yaml - Standard Helm templating
stringData:
  db-password: {{ .Values.secrets.database.password }}
  api-key: {{ .Values.secrets.api.key }}
```

**Why this matters**: Your Helm templates remain **portable** and vendor-agnostic. The same chart works with any tool - only the values file changes!

**Secrets are stored in**: HashiCorp Vault (configured in Harness)

**Perfect for**: Teams using Helm

[→ View complete Helm example with setup instructions](./helm-values-example/)

---

### 📁 Bonus Sample 3: Same Manifests for GitOps AND CD Pipeline ([`/shared-manifests-example`](./shared-manifests-example/))

**Important discovery**: You **do NOT need separate manifests** for GitOps vs CD Pipelines!

**What this shows**:
- The **exact same Git repository and manifests** work with both:
  - ✅ GitOps Application (Argo CD sync)
  - ✅ CD Pipeline (traditional Harness deployment)
- Secret expressions resolve correctly in both scenarios
- No duplication needed!

**Where are the secret expressions?**  
→ **[`shared-manifests-example/manifests/secret.yaml`](./shared-manifests-example/manifests/secret.yaml)** - Same expressions work for both deployment methods

**Perfect for**: Teams asking "Do I need separate manifests for GitOps?" Answer: **No!**

[→ View shared manifests example](./shared-manifests-example/)

---

### 🎯 Which Example Should I Use?

| Your Situation | Use This Sample |
|----------------|-----------------|
| Using plain Kubernetes YAML | [Sample 1: Simple Example](./simple-example/) |
| Using Helm charts | [Sample 2: Helm Values Example](./helm-values-example/) |
| Want to use both GitOps and CD Pipeline | [Sample 3: Shared Manifests](./shared-manifests-example/) |
| New to the feature | Start with [Sample 1](./simple-example/), it's the simplest |

---

### 📍 Quick Reference: Where Are Secret Expressions?

**For the simple example**:
- File: `simple-example/secret.yaml`
- Location: In the `stringData` field of the Secret resource
- Secrets stored in: HashiCorp Vault

**For the Helm example**:
- File: `helm-values-example/values.yaml` (expressions here)
- File: `helm-values-example/templates/secret.yaml` (uses the values)
- Secrets stored in: HashiCorp Vault

**For all examples**:
- The secret expressions reference secrets stored in **HashiCorp Vault**
- The Vault integration is configured in Harness as your Secret Manager
- The Harness GitOps plugin resolves the expressions during manifest rendering

---

> 💡 **Key Takeaway**: Secret expressions work the same way whether you're using plain manifests, Helm charts, GitOps, or CD Pipelines. Pick the pattern that matches your workflow!

---

## Setup Instructions

### Step 1: Create Secrets in Harness

1. Navigate to your Harness account/org/project
2. Go to **Account/Org/Project Settings** → **Secrets**
3. Click **+ New Secret** → **Text**
4. Configure your secret:
   - **Secret Name**: Give it a descriptive name (e.g., `apiKey`, `dbPassword`)
   - **Identifier**: This is what you'll use in expressions (e.g., `apiKey`, `db_password`)
   - **Secret Manager**: Choose **HashiCorp Vault** or **Harness Secret Manager**
   - **Secret Value**: Enter your sensitive value

5. Repeat for all required secrets at the appropriate scope:
   - Use **Account** scope for secrets shared across all orgs/projects
   - Use **Org** scope for secrets shared within an organization
   - Use **Project** scope for project-specific secrets

### Step 2: Prepare Your Manifests

1. Choose an example from this repository that matches your use case
2. Update the secret expressions to match your Harness secret identifiers
3. Commit your manifests to your Git repository

**Example**:
```yaml
# If your Harness secret identifier is "prod_api_key" at org level
apiVersion: v1
kind: Secret
metadata:
  name: api-credentials
stringData:
  api-key: <+secrets.getValue("org.prod_api_key")>
```

### Step 3: Create GitOps Application

1. Go to **Deployments** → **GitOps** → **Applications**
2. Click **+ New Application**
3. Configure the application:
   - **Application Name**: Choose a name
   - **Agent**: Select your GitOps agent (must have the Harness plugin enabled)
   - **Repository**: Point to your Git repo
   - **Path**: Specify the path to your manifests
   - **Cluster**: Select target cluster
   - **Namespace**: Specify namespace

4. Click **Create**

### Step 4: Review and Sync

1. After creation, Argo CD will render your manifests
2. Click on the application to view details
3. Go to **App Diff** tab to see the diff with secret expressions resolved
   - ✅ You should see secret values are masked/obfuscated
   - ✅ The diff shows changes but not the actual secret values

4. Click **Sync** to apply the manifests to your cluster

5. Verify in your cluster:
```bash
kubectl get secrets -n <namespace>
kubectl describe secret <secret-name> -n <namespace>
# Values should be shown as [Redacted] or base64 encoded
```

### Step 5: Using Secrets in Your Application

Reference the Kubernetes secret in your Deployment/Pod:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      containers:
      - name: app
        image: my-app:latest
        env:
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: api-credentials  # Reference the Secret resource
              key: api-key           # Reference the specific key
```

---

## Important Notes and Gotchas

### 🔐 Security Considerations

1. **Argo CD Redis Cache**: 
   - Argo CD caches manifests (including resolved secrets) in its Redis instance
   - **Mitigation**: Implement network policies to restrict access to Argo CD components
   - **Best Practice**: Consider running Argo CD on a dedicated cluster

2. **Secret Visibility**:
   - Users with direct access to Argo CD's Redis database can see resolved secrets
   - This is an Argo CD limitation, not specific to Harness
   - **Recommendation**: Do not give users direct access to Argo CD's database

3. **UI Masking**:
   - Secret values are automatically masked in the Harness UI
   - Kubernetes also masks values in Secret resources
   - However, users with kubectl access and proper RBAC can still retrieve secret values (this is standard Kubernetes behavior)

### ⚠️ Common Pitfalls

1. **Wrong Resource Type**:
   ```yaml
   # ❌ WRONG - Won't work in ConfigMap
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: config
   data:
     api-key: <+secrets.getValue("apiKey")>  # Will NOT be resolved!
   ```

   ```yaml
   # ✅ CORRECT - Works in Secret
   apiVersion: v1
   kind: Secret
   metadata:
     name: secret
   stringData:
     api-key: <+secrets.getValue("apiKey")>  # Will be resolved
   ```

2. **Wrong Scope Prefix**:
   ```yaml
   # If secret is at org level, must use "org." prefix
   # ❌ WRONG
   password: <+secrets.getValue("dbPassword")>
   
   # ✅ CORRECT
   password: <+secrets.getValue("org.dbPassword")>
   ```

3. **Agent Not Configured**:
   - Ensure the Harness plugin is enabled on your GitOps agent
   - Without it, secret expressions won't be resolved

4. **Unsupported Secret Manager**:
   - Only HashiCorp Vault and Harness Secret Manager are currently supported
   - AWS Secrets Manager, Azure Key Vault, GCP Secret Manager are NOT supported

### 🎯 Best Practices

1. **Use Appropriate Scopes**:
   - Account-level: For secrets shared across entire Harness account
   - Org-level: For secrets shared within an organization
   - Project-level: For application-specific secrets

2. **Naming Conventions**:
   - Use clear, descriptive secret identifiers
   - Consider prefixing with environment: `prod_api_key`, `dev_db_password`

3. **Secret Rotation**:
   - Update secret values in Harness
   - Manually sync the application in Argo CD to pick up new values
   - Consider automation for regular rotation

4. **Testing**:
   - Test in dev/staging environments first
   - Verify secret resolution in App Diff before syncing
   - Confirm application functionality after sync

---

## Troubleshooting

### Secret Expression Not Resolved

**Problem**: Expression appears as literal text in the cluster

**Solutions**:
1. Verify feature flag `CDS_GITOPS_SECRET_RESOLUTION_ENABLED` is enabled
2. Confirm Harness plugin is enabled on the GitOps agent
3. Check that the secret exists in Harness with the correct identifier
4. Ensure you're using the correct scope prefix (account./org./no prefix)
5. Verify the expression is in a `kind: Secret` resource, not ConfigMap or Deployment

### Application Fails to Sync

**Problem**: Sync fails with error about secret resolution

**Solutions**:
1. Check that the secret identifier matches exactly (case-sensitive)
2. Verify you have RBAC permissions to access the secret in Harness
3. Confirm the secret manager (Vault/Harness SM) is configured correctly
4. Check Argo CD logs for detailed error messages:
   ```bash
   kubectl logs -n <argocd-namespace> -l app.kubernetes.io/name=argocd-repo-server
   ```

### Secret Value Not Updating

**Problem**: Updated secret in Harness but application still uses old value

**Solutions**:
1. Manually sync the application in Argo CD
2. Argo CD caches rendered manifests - force a refresh:
   - Click **Refresh** in the Argo CD UI
   - Or run: `argocd app get <app-name> --refresh`

### Permission Denied Errors

**Problem**: Cannot resolve secrets due to permission errors

**Solutions**:
1. Verify your Harness account has permissions to read the secrets
2. Check that the GitOps agent has proper service account permissions
3. Ensure the secret scope matches your project/org structure

---

## Screenshots

Below are key screenshots demonstrating the feature in action:

### 1. Creating Secrets in Harness
![Create Secret in Harness](../static/gitops-secret-create.png)
*Creating a new secret at the project level in Harness*

### 2. GitOps Agent Configuration
![Enable Harness Plugin](../static/gitops-agent-plugin.png)
*Enabling the Harness Plugin during agent installation*

### 3. Application with Secret Expressions
![Application Manifest](../static/gitops-secret-manifest.png)
*Manifest file with secret expressions in Git repository*

### 4. Viewing App Diff with Masked Secrets
![App Diff View](../static/gitops-secret-diff.png)
*App Diff showing secret expressions resolved but values masked*

### 5. Synced Application
![Synced Application](../static/gitops-secret-synced.png)
*Successfully synced application with secrets*

### 6. Secret in Kubernetes Cluster
![Kubernetes Secret](../static/gitops-secret-k8s.png)
*Verifying the secret exists in the Kubernetes cluster*

---

## Additional Resources

- **Official Documentation**: https://developer.harness.io/docs/continuous-delivery/gitops/application/manage-gitops-applications#harness-secret-expressions-in-application-manifests
- **Argo CD Secret Management**: https://argo-cd.readthedocs.io/en/stable/operator-manual/secret-management/
- **Harness GitOps Documentation**: https://developer.harness.io/docs/continuous-delivery/gitops/
- **Feature Demo Video**: [Available in Harness Community]

---

## Support

If you encounter issues:
1. Check the troubleshooting section above
2. Review Argo CD and GitOps agent logs
3. Contact Harness Support with:
   - Feature flag status
   - Agent version and configuration
   - Error messages from logs
   - Example manifests (redacted)

---

## Future Enhancements

The following features are planned for future releases:

- ⏳ Service Variable Expressions in manifests
- ⏳ Environment Variable Expressions in manifests
- ⏳ Project Variable Expressions in manifests
- ⏳ Support for additional secret managers (AWS, Azure, GCP)

Expected: Q4 2024 / Q1 2025

---

## Contributing

This is a community repository. If you have improvements or additional examples:
1. Fork the repository
2. Create your feature branch
3. Add your examples with documentation
4. Submit a pull request

---

**Last Updated**: December 2025  
**Feature Flag**: `CDS_GITOPS_SECRET_RESOLUTION_ENABLED`  
**Minimum Agent Version**: Check Harness documentation for compatibility


