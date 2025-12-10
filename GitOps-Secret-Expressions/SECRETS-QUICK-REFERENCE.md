# 🔑 Quick Reference: Project-Level Secrets

## All Secrets to Create in Harness

**Location**: `Project Settings` → `Secrets` → `+ New Secret` → `Text`

### Required Secrets (5 total)

| # | Identifier | Name | Example Value | Secret Manager |
|---|------------|------|---------------|----------------|
| 1 | `apiKey` | API Key | demo-api-key-12345 | HashiCorp Vault |
| 2 | `dbPassword` | Database Password | demo-db-pass-789 | HashiCorp Vault |
| 3 | `dbUsername` | Database Username | app_user | HashiCorp Vault |
| 4 | `appSecret` | Application Secret | demo-app-secret-abc | HashiCorp Vault |
| 5 | `redisPassword` | Redis Password | demo-redis-pass-xyz | HashiCorp Vault |

### Test Secret (1 total)

| # | Identifier | Name | Example Value | Secret Manager |
|---|------------|------|---------------|----------------|
| 6 | `testSecret` | Test Secret | hello-world-123 | HashiCorp Vault |

---

## Expression Syntax (Project-Level)

```yaml
# Simple syntax - no prefix needed!
api-key: <+secrets.getValue("apiKey")>
db-password: <+secrets.getValue("dbPassword")>
db-username: <+secrets.getValue("dbUsername")>
app-secret: <+secrets.getValue("appSecret")>
redis-password: <+secrets.getValue("redisPassword")>
```

---

## Where Each Secret is Used

```
simple-example/secret.yaml
├── apiKey ✓
├── dbPassword ✓
├── dbUsername ✓
├── appSecret ✓
└── redisPassword ✓

helm-values-example/values.yaml
├── apiKey ✓
├── dbPassword ✓
├── dbUsername ✓
├── appSecret ✓
└── redisPassword ✓

shared-manifests-example/manifests/secret.yaml
├── apiKey ✓
├── dbPassword ✓
├── dbUsername ✓
└── appSecret ✓

test-permission/secret-test.yaml
└── testSecret ✓
```

---

## Checklist

### Before Creating Applications:

- [ ] HashiCorp Vault connector configured in Harness
- [ ] Feature flag `CDS_GITOPS_SECRET_RESOLUTION_ENABLED` enabled
- [ ] GitOps agent has Harness plugin enabled
- [ ] All 5 main secrets created (apiKey, dbPassword, dbUsername, appSecret, redisPassword)
- [ ] testSecret created (for testing)
- [ ] All secrets use HashiCorp Vault as Secret Manager
- [ ] GitOps agent service account has Secret View & Access permissions

### After Creating Secrets:

- [ ] Test with permission-test app first
- [ ] If successful, deploy simple-secrets-app
- [ ] Then deploy helm-secrets-app
- [ ] Finally deploy shared-manifests-app

---

## Quick Copy-Paste for Secret Creation

```
Secret 1:
Name: API Key
Identifier: apiKey
Secret Manager: HashiCorp Vault
Value: <your-api-key>

Secret 2:
Name: Database Password
Identifier: dbPassword
Secret Manager: HashiCorp Vault
Value: <your-db-password>

Secret 3:
Name: Database Username
Identifier: dbUsername
Secret Manager: HashiCorp Vault
Value: <your-db-username>

Secret 4:
Name: Application Secret
Identifier: appSecret
Secret Manager: HashiCorp Vault
Value: <your-app-secret>

Secret 5:
Name: Redis Password
Identifier: redisPassword
Secret Manager: HashiCorp Vault
Value: <your-redis-password>

Test Secret:
Name: Test Secret
Identifier: testSecret
Secret Manager: HashiCorp Vault
Value: hello-world-123
```

---

**Total Time to Create**: ~5-10 minutes  
**Scope**: Project Level  
**Secret Manager**: HashiCorp Vault

