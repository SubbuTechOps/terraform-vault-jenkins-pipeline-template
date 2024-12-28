# Complete Guide: HashiCorp Vault Integration with Jenkins Pipeline for AWS Credentials

## Table of Contents
1. Prerequisites
2. HashiCorp Vault Setup
3. AWS Credentials Configuration in Vault
4. Jenkins Configuration
5. Pipeline Implementation
6. Testing and Validation
7. Troubleshooting

## 1. Prerequisites

### Required Software
- HashiCorp Vault
- Jenkins
- AWS CLI
- jq (JSON processor)

### Required Jenkins Plugins
- Pipeline
- Credentials Plugin
- HashiCorp Vault Plugin

## 2. HashiCorp Vault Setup

### 2.1 Install and Initialize Vault
```bash
# Start Vault server in dev mode (for testing)
vault server -dev

# In production, use proper configuration
vault server -config=/path/to/config.hcl

# Initialize Vault (in production)
vault operator init

# Unseal Vault (in production)
vault operator unseal
```

### 2.2 Enable AppRole Authentication
```bash
# Enable AppRole auth method
vault auth enable approle

# Create a policy for AWS access
cat << EOF > aws-policy.hcl
path "kv/data/aws/*" {
  capabilities = ["read"]
}
EOF

vault policy write aws-policy aws-policy.hcl

# Create AppRole
vault write auth/approle/role/jenkins-role \
    token_policies="aws-policy" \
    token_ttl=1h \
    token_max_ttl=4h

# Get Role ID
vault read auth/approle/role/jenkins-role/role-id
# Store the Role ID

# Generate Secret ID
vault write -f auth/approle/role/jenkins-role/secret-id
# Store the Secret ID
```

## 3. AWS Credentials Configuration in Vault

### 3.1 Enable KV Secrets Engine
```bash
# Enable KV version 2
vault secrets enable -version=2 kv

# Store AWS credentials
vault kv put kv/aws/terraform \
    access_key=YOUR_AWS_ACCESS_KEY \
    secret_key=YOUR_AWS_SECRET_KEY
```

### 3.2 Verify Credentials Storage
```bash
# Read back the secrets
vault kv get kv/aws/terraform
```

## 4. Jenkins Configuration

### 4.1 Configure Jenkins Credentials
1. Navigate to Jenkins > Manage Jenkins > Manage Credentials
2. Add new credentials:
   - Kind: Secret text
   - ID: vault-role-id
   - Secret: [Role ID from step 2.2]

   Add another:
   - Kind: Secret text
   - ID: vault-secret-id
   - Secret: [Secret ID from step 2.2]

### 4.2 Configure Vault URL
1. Add to Jenkins configuration:
   - VAULT_ADDR environment variable
   - Vault server certificate (if using HTTPS)

## 5. Pipeline Implementation

### 5.1 Jenkins Pipeline Structure
```groovy
// Complete Pipeline code provided in previous response
```

### 5.2 Required Files Structure
```
project/
├── Jenkinsfile
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── backend.tf
│   └── environments/
│       ├── dev.tfvars
│       ├── staging.tfvars
│       └── prod.tfvars
```

## 6. Testing and Validation

### 6.1 Vault Connection Test
```bash
# Test Vault authentication
curl -X POST \
    -d '{"role_id":"ROLE_ID","secret_id":"SECRET_ID"}' \
    ${VAULT_ADDR}/v1/auth/approle/login
```

### 6.2 Pipeline Test
1. Create test branch
2. Modify test environment
3. Run pipeline with 'plan' action
4. Verify credential retrieval
5. Verify Terraform execution

## 7. Troubleshooting

### 7.1 Common Issues and Solutions

#### Vault Authentication Failures
```bash
# Check Vault status
vault status

# Verify role
vault read auth/approle/role/jenkins-role

# Check policy
vault policy read aws-policy
```

#### Jenkins Pipeline Issues
```groovy
// Add debug logging
stage('Debug') {
    steps {
        script {
            def vaultToken = sh(
                script: """
                    curl -s -X POST \
                    -d '{"role_id":"${ROLE_ID}"}' \
                    ${VAULT_ADDR}/v1/auth/approle/login
                """,
                returnStdout: true
            )
            echo "Vault Response: ${vaultToken}"
        }
    }
}
```

### 7.2 Security Best Practices

1. Credential Rotation
```bash
# Rotate Secret ID
vault write -f auth/approle/role/jenkins-role/secret-id

# Update Jenkins credential with new Secret ID
```

2. Audit Logging
```bash
# Enable audit logging
vault audit enable file file_path=/var/log/vault/audit.log

# Monitor access
tail -f /var/log/vault/audit.log
```

## Security Considerations

1. Production Setup:
   - Use TLS for Vault communications
   - Enable audit logging
   - Implement proper backup strategy
   - Use proper ACLs and policies

2. Credential Management:
   - Rotate credentials regularly
   - Use least privilege principle
   - Monitor access logs
   - Implement credential revocation process

3. Pipeline Security:
   - Mask sensitive output
   - Clean workspace after execution
   - Implement proper error handling
   - Use secure Jenkins configuration

## Monitoring and Maintenance

1. Regular Tasks:
   - Rotate credentials
   - Update policies
   - Review access logs
   - Verify backup procedures

2. Health Checks:
   - Monitor Vault status
   - Check Jenkins-Vault connectivity
   - Verify policy enforcement
   - Test credential retrieval

Remember to replace placeholder values with your actual configuration details and adapt the security measures according to your organization's requirements.
