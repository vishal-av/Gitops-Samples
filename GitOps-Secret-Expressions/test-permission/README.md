# Permission Test

Use this minimal example to test if the GitOps agent can access secrets.

## Setup

1. **Create a simple test secret in Harness**:
   - Go to **Project Settings** → **Secrets**
   - Click **+ New Secret** → **Text**
   - Name: `Test Secret`
   - Identifier: `testSecret`
   - Secret Manager: Harness Secret Manager
   - Value: `hello-world-123`

2. **Create a test GitOps application**:
   - Name: `permission-test`
   - Repository: Gitops Samples
   - Path: `GitOps-Secret-Expressions/test-permission`
   - Cluster: Your cluster
   - Namespace: `default`

3. **Try to sync**

## Expected Results

### ✅ If Permissions Are Correct:
- App syncs successfully
- Secret `permission-test` is created in cluster
- Expression is resolved

### ❌ If Permissions Issue:
- Error: `permission denied`
- Error: `non-200 response: 500`

## Fix Permissions

If you get permission denied:

1. **Find GitOps Agent Service Account**:
   - Project Settings → Access Control → Service Accounts
   - Find: `gitops-agent-<your-agent-name>`

2. **Grant Secret Access**:
   - Click on the service account
   - Role Bindings → + New Role Binding
   - Role: Choose one with **Secret View** and **Secret Access** permissions
   - Scope: Project (or Account/Org if using those scopes)

3. **Wait 1-2 minutes** for permissions to propagate

4. **Retry sync**

## Clean Up

After testing, delete:
```bash
kubectl delete secret permission-test -n default
```

And delete the test app in Harness UI.

