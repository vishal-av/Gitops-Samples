# 🔑 Required Project-Level Secrets

All examples now use **PROJECT-LEVEL secrets only** for simplicity!

## Where to Create These Secrets

**Harness UI**: `Project Settings` → `Secrets` → `+ New Secret` → `Text`

---

## 📋 Secrets to Create

### 1. **apiKey**
```
Name: API Key
Identifier: apiKey  ⭐ (must match exactly!)
Secret Manager: HashiCorp Vault (or Harness Secret Manager)
Value: demo-api-key-12345
Description: API key for external services
```

### 2. **dbPassword**
```
Name: Database Password
Identifier: dbPassword  ⭐
Secret Manager: HashiCorp Vault
Value: demo-db-pass-secret-789
Description: Database password for application
```

### 3. **dbUsername**
```
Name: Database Username
Identifier: dbUsername  ⭐
Secret Manager: HashiCorp Vault
Value: app_user
Description: Database username for application
```

### 4. **appSecret**
```
Name: Application Secret
Identifier: appSecret  ⭐
Secret Manager: HashiCorp Vault
Value: demo-app-secret-key-abc123
Description: Application secret key for encryption
```

### 5. **redisPassword**
```
Name: Redis Password
Identifier: redisPassword  ⭐
Secret Manager: HashiCorp Vault
Value: demo-redis-pass-xyz456
Description: Redis/Cache password
```

### 6. **testSecret** (for testing only)
```
Name: Test Secret
Identifier: testSecret  ⭐
Secret Manager: HashiCorp Vault
Value: hello-world-test-123
Description: Test secret for permission validation
```

---

## ⚡ Quick Create Checklist

- [ ] **apiKey** - API key for external services
- [ ] **dbPassword** - Database password
- [ ] **dbUsername** - Database username
- [ ] **appSecret** - Application secret key
- [ ] **redisPassword** - Redis/Cache password
- [ ] **testSecret** - Test secret (for permission-test app)

---

## 🎯 Step-by-Step Creation

### Step 1: Navigate to Project Secrets
```
1. Go to your Harness Project
2. Click "Settings" (⚙️) in left sidebar
3. Click "Secrets" under Security
4. Click "+ New Secret" button
5. Select "Text"
```

### Step 2: Create Each Secret

For each secret above, fill in:

```
┌─────────────────────────────────────────┐
│ Create Secret                           │
├─────────────────────────────────────────┤
│                                         │
│ Secret Name: API Key                    │
│ Identifier: apiKey       ⭐ IMPORTANT! │
│                                         │
│ Select Secret Manager:                  │
│ ○ Harness Secret Manager                │
│ ● HashiCorp Vault        (recommended)  │
│                                         │
│ Secret Value:                           │
│ [••••••••••••••••••]                   │
│                                         │
│ Description (optional):                 │
│ API key for external services           │
│                                         │
│ [Cancel]  [Save] ✓                      │
└─────────────────────────────────────────┘
```

### Step 3: Verify All Secrets Created

After creating all secrets, you should see:

```
Project Settings → Secrets

✓ apiKey
✓ appSecret  
✓ dbPassword
✓ dbUsername
✓ redisPassword
✓ testSecret
```

---

## 🔍 Why Project-Level?

**Advantages of Project-Level Secrets**:
- ✅ **Simpler syntax**: No `account.` or `org.` prefix needed
- ✅ **Easier permissions**: Just need project-level access
- ✅ **Better isolation**: Secrets scoped to specific project
- ✅ **Faster troubleshooting**: All secrets in one place

**Expression Syntax**:
```yaml
# Project-level (what we're using now)
password: <+secrets.getValue("dbPassword")>

# vs Organization-level (more complex)
password: <+secrets.getValue("org.dbPassword")>

# vs Account-level (most complex)
password: <+secrets.getValue("account.dbPassword")>
```

---

## 🧪 Testing Secrets

### Test 1: Verify Secrets Exist
```bash
# Via Harness UI
Project Settings → Secrets → Should see all 6 secrets
```

### Test 2: Test with Permission Test App
```yaml
1. Create the testSecret (see above)
2. Deploy test app:
   - Name: permission-test
   - Path: GitOps-Secret-Expressions/test-permission
   - Namespace: devrel-pg
3. Sync and check for errors
4. If successful, proceed to main examples
```

### Test 3: Deploy Simple Example
```yaml
1. Ensure all 5 main secrets created
2. Deploy app:
   - Name: simple-secrets-app
   - Path: GitOps-Secret-Expressions/simple-example
   - Namespace: devrel-pg
3. Sync and verify
```

---

## 🔒 Secret Values (for Demo)

You can use these demo values for testing:

```bash
# Safe demo values (NOT for production!)
apiKey: "demo_api_key_12345_test"
dbPassword: "demo_db_password_secret_789"
dbUsername: "demo_app_user"
appSecret: "demo_app_secret_key_abc123xyz"
redisPassword: "demo_redis_password_xyz456"
testSecret: "hello_world_test_123"
```

**⚠️ For Production**: Use strong, unique values from your actual systems!

---

## 📊 Secrets Usage Matrix

| Secret Identifier | Simple Example | Helm Example | Shared Example | Test |
|-------------------|----------------|--------------|----------------|------|
| **apiKey** | ✓ | ✓ | ✓ | |
| **dbPassword** | ✓ | ✓ | ✓ | |
| **dbUsername** | ✓ | ✓ | ✓ | |
| **appSecret** | ✓ | ✓ | ✓ | |
| **redisPassword** | ✓ | ✓ | | |
| **testSecret** | | | | ✓ |

---

## 🚨 Common Issues

### Issue: "Secret not found"
**Cause**: Identifier doesn't match exactly
**Fix**: Check identifier is exact (case-sensitive):
- ✅ `apiKey` (correct)
- ❌ `ApiKey` (wrong - capital A)
- ❌ `api_key` (wrong - underscore)

### Issue: "Permission denied"
**Cause**: Service account doesn't have access to secrets
**Fix**: 
1. Go to Project Settings → Access Control → Service Accounts
2. Find GitOps agent service account
3. Add role with "Secret View" and "Secret Access" permissions

### Issue: "Wrong secret value"
**Cause**: Wrong secret manager or secret updated
**Fix**:
1. Verify all secrets use same secret manager (HashiCorp Vault)
2. Check secret values in Harness UI
3. Re-sync application after updating secrets

---

## ✅ Verification Script

Run this to verify all secrets are created:

```bash
# This is pseudo-code - check in Harness UI
# Go to Project Settings → Secrets and verify:

Required Secrets:
1. apiKey ...................... [ ]
2. dbPassword .................. [ ]
3. dbUsername .................. [ ]
4. appSecret ................... [ ]
5. redisPassword ............... [ ]
6. testSecret (for testing) .... [ ]

All secrets using HashiCorp Vault? [ ]
```

---

## 🎯 Next Steps

Once all secrets are created:

1. ✅ **Test with permission-test app** first
2. ✅ **Deploy simple-secrets-app**
3. ✅ **Deploy helm-secrets-app**
4. ✅ **Deploy shared-manifests-app**

---

**Last Updated**: December 2025  
**Secret Scope**: Project Level Only  
**Total Secrets Required**: 5 (+ 1 for testing)

