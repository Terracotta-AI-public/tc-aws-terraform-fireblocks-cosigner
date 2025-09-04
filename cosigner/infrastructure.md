# Cosigner Module Infrastructure Documentation

## Overview

The Cosigner module implements a secure, AWS Nitro Enclave-based infrastructure for Fireblocks Multi-Party Computation (MPC) cryptographic signing operations. This module provides hardware-level security isolation for sensitive key material and signing operations.

## Architecture

### Core Components
- **AWS Nitro Enclave EC2 Instance**: Isolated compute environment for cryptographic operations
- **KMS Customer Master Key**: Encryption key with attestation-based access control
- **S3 Bucket**: Secure storage for key shares and audit logs
- **IAM Roles & Policies**: Fine-grained access control
- **Security Groups**: Network-level access restrictions

### Security Model
```
┌─────────────────────────────────────┐
│         Parent EC2 Instance         │
│  ┌─────────────────────────────┐    │
│  │    AWS Nitro Enclave        │    │
│  │  ┌─────────────────────┐    │    │
│  │  │ MPC Signing Logic   │    │    │
│  │  │ PCR8 Attestation    │    │    │
│  │  └─────────────────────┘    │    │
│  └─────────────────────────────┘    │
└─────────────────────────────────────┘
           ↓ Attestation
    ┌──────────────┐
    │   AWS KMS    │
    └──────────────┘
           ↓
    ┌──────────────┐
    │   S3 Bucket  │
    └──────────────┘
```

## EC2 Instance Configuration

### Instance Specifications
| Parameter | Value | Rationale |
|-----------|-------|-----------|
| **Instance Type** | `c5.xlarge` | 4 vCPUs, 8 GiB RAM - Optimal for enclave operations |
| **AMI** | Amazon Linux 2023 (latest) | Native Nitro Enclave support |
| **Architecture** | x86_64 | Required for enclave compatibility |
| **Instance Name** | `nitro-mainnet-01` | Production mainnet identifier |

### Nitro Enclave Settings
```hcl
enclave_options {
  enabled = true
}
```
- **CPU Allocation**: Dedicated vCPUs for enclave (configurable)
- **Memory Allocation**: Isolated memory region
- **Attestation**: PCR-based integrity verification
- **Network**: No direct network access (vsock communication only)

### Storage Configuration
| Component | Specification | Purpose |
|-----------|--------------|---------|
| **Root Volume** | 100 GB GP3 | OS and application storage |
| **Volume Type** | GP3 | Balanced performance/cost |
| **IOPS** | 3000 (baseline) | Standard GP3 performance |
| **Throughput** | 125 MB/s | Default GP3 throughput |
| **Encryption** | Default (optional KMS) | At-rest encryption |
| **Delete on Termination** | Yes | Clean resource management |

### Metadata Service (IMDSv2)
```hcl
metadata_options {
  http_tokens               = "required"
  http_put_response_hop_limit = 2
  http_endpoint             = "enabled"
}
```
- **Security**: Enforces IMDSv2 for protection against SSRF attacks
- **Token Requirement**: Session-based authentication
- **Hop Limit**: Allows 2 hops for container environments

### Instance Profile
- **Name**: `nitro-mainnet-ec2-role-profile-{random_suffix}`
- **Role Association**: Links IAM role to EC2 instance
- **Credential Provider**: Automatic credential rotation via STS

## IAM Configuration

### Role Structure

#### Primary EC2 Role
**Name**: `nitro-mainnet-ec2-role-{random_suffix}`

**Trust Policy**:
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "ec2.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
```

### Permission Policies

#### 1. S3 Access Policy
**Purpose**: Secure storage access for key shares and audit logs

| Action | Resource | Purpose |
|--------|----------|---------|
| `s3:ListAllMyBuckets` | `*` | Bucket discovery |
| `s3:ListBucket` | Specific bucket ARN | Object enumeration |
| `s3:GetBucketLocation` | Specific bucket ARN | Region identification |
| `s3:PutObject` | Bucket objects (`/*`) | Write key shares |
| `s3:GetObject` | Bucket objects (`/*`) | Read key shares |
| `s3:DeleteObject` | Bucket objects (`/*`) | Cleanup operations |
| `s3:PutObjectAcl` | Bucket objects (`/*`) | ACL management |
| `s3:GetObjectAcl` | Bucket objects (`/*`) | ACL verification |

#### 2. KMS Access Policy
**Purpose**: Cryptographic operations on key material

| Action | Purpose |
|--------|---------|
| `kms:Encrypt` | Encrypt sensitive data |
| `kms:Decrypt` | Decrypt stored secrets |
| `kms:GenerateDataKey` | Create data encryption keys |
| `kms:GenerateDataKeyPair` | Asymmetric key generation |
| `kms:GenerateDataKeyWithoutPlaintext` | Secure key generation |
| `kms:GenerateDataKeyPairWithoutPlaintext` | Secure asymmetric keys |
| `kms:GenerateRandom` | Cryptographic randomness |
| `kms:GetKeyPolicy` | Policy verification |

#### 3. Systems Manager Access
**Attached Policy**: `arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore`

**Capabilities**:
- Session Manager access
- Parameter Store read
- Patch management
- CloudWatch metrics

### S3 Bucket Policy

**Deny-by-Default Security Model**:
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Deny",
    "Principal": {"AWS": "*"},
    "Action": [
      "s3:PutObject",
      "s3:GetObject",
      "s3:DeleteObject",
      "s3:ListBucket",
      "s3:ListBucketMultipartUploads",
      "s3:AbortMultipartUpload",
      "s3:GetObjectAcl",
      "s3:PutObjectAcl",
      "s3:RestoreObject"
    ],
    "Resource": [
      "arn:aws:s3:::bucket-name",
      "arn:aws:s3:::bucket-name/*"
    ],
    "Condition": {
      "ArnNotEquals": {
        "aws:PrincipalArn": "arn:aws:iam::account:role/role-name"
      }
    }
  }]
}
```

## KMS Configuration

### Key Specifications
| Parameter | Value | Purpose |
|-----------|-------|---------|
| **Key Type** | Symmetric | AES-256 encryption |
| **Key Usage** | ENCRYPT_DECRYPT | Data encryption operations |
| **Key Spec** | SYMMETRIC_DEFAULT | Standard symmetric key |
| **Multi-Region** | No | Regional isolation |
| **Rotation** | Manual | Controlled key lifecycle |

### Key Policy

#### 1. Administrative Access
```json
{
  "Sid": "Enable IAM User Permissions",
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::account-id:root"
  },
  "Action": "kms:*",
  "Resource": "*"
}
```

#### 2. Enclave-Specific Access (PCR8 Attestation)
```json
{
  "Sid": "Enable enclave data processing for specific role",
  "Effect": "Allow",
  "Actions": [
    "kms:Decrypt",
    "kms:Encrypt",
    "kms:GenerateDataKey",
    "kms:GenerateDataKeyPair",
    "kms:GenerateDataKeyWithoutPlaintext",
    "kms:GenerateDataKeyPairWithoutPlaintext",
    "kms:GenerateRandom",
    "kms:GetKeyPolicy"
  ],
  "Resource": "*",
  "Principal": {
    "AWS": "arn:aws:iam::account:role/nitro-mainnet-ec2-role"
  },
  "Condition": {
    "StringEqualsIgnoreCase": {
      "kms:RecipientAttestation:PCR8": "da1d9eca20ce98ab4fdbc51f8e5a2307fd4c61829b7d8bff40976cd6676862c8f3476ff4bdd0f65ecf4a48d6eb3099a8"
    }
  }
}
```

### PCR8 Attestation Details
- **PCR8 Value**: `da1d9eca20ce98ab4fdbc51f8e5a2307fd4c61829b7d8bff40976cd6676862c8f3476ff4bdd0f65ecf4a48d6eb3099a8`
- **Purpose**: Verifies enclave image integrity
- **Validation**: KMS checks attestation document before allowing operations
- **Security**: Prevents unauthorized enclave images from accessing keys

## S3 Storage

### Bucket Configuration
| Parameter | Value | Purpose |
|-----------|-------|---------|
| **Bucket Name** | `nitro-mainnet-bucket-{random_suffix}` | Unique identifier |
| **Versioning** | Disabled (default) | Simple object management |
| **Encryption** | SSE-KMS | At-rest encryption |
| **Public Access** | Blocked | Security requirement |
| **Object Lock** | Disabled | Not required for use case |

### Storage Structure
```
nitro-mainnet-bucket-{suffix}/
├── keyshares/
│   ├── shard_1.enc
│   ├── shard_2.enc
│   └── shard_3.enc
├── attestation/
│   ├── {timestamp}_attestation.json
│   └── pcr_values.json
├── audit/
│   ├── {date}/
│   │   ├── signing_operations.log
│   │   └── access_logs.json
└── config/
    └── enclave_config.json
```

### Access Control
- **Write**: Restricted to Nitro EC2 role
- **Read**: Restricted to Nitro EC2 role
- **Delete**: Restricted to specific objects
- **List**: Allowed for authorized role only

## Security Group Configuration

### Security Group: `nitro-instance-sg`

#### Inbound Rules

| Port | Protocol | Source CIDR | Service | Purpose |
|------|----------|-------------|---------|---------|
| 8080 | TCP | 12.0.0.0/16 | HTTP API | RESTful API endpoint for signing requests |
| 4001 | TCP | 12.0.0.0/16 | Internal Service | Inter-service communication |
| 50051 | TCP | 12.0.0.0/16 | gRPC | High-performance RPC for MPC operations |

#### Outbound Rules

| Port | Protocol | Destination | Purpose |
|------|----------|------------|---------|
| All | All | 0.0.0.0/0 | Unrestricted egress for updates and external APIs |

### Network Security Considerations
- **Internal CIDR**: 12.0.0.0/16 restricts access to VPC internal network
- **No SSH**: Direct SSH access disabled for security
- **Session Manager**: Access via AWS Systems Manager
- **Stateful Rules**: Return traffic automatically allowed

## Terraform Variables

### Required Variables
| Variable | Type | Description | Example |
|----------|------|-------------|---------|
| `vpc_id` | string | Target VPC for deployment | `vpc-0123456789abcdef` |
| `subnet_id` | string | Subnet for EC2 instance | `subnet-0123456789abcdef` |

### Optional Variables
| Variable | Default | Description | Customization |
|----------|---------|-------------|---------------|
| `aws_region` | `us-west-2` | AWS deployment region | Any valid region |
| `aws_account_id` | `590184059249` | AWS account identifier | Your account ID |
| `ami_name_pattern` | `al2023-ami-*-x86_64` | AMI filter pattern | Custom AMI pattern |
| `ami_owner` | `amazon` | AMI owner account | `self` for custom |
| `internal_cidr_blocks` | `["12.0.0.0/16"]` | Internal network CIDRs | Your VPC CIDRs |
| `key_name` | `cosigner-nitro-prod` | EC2 key pair name | Your key pair |

## Outputs

### Exported Values
| Output | Type | Description | Usage |
|--------|------|-------------|-------|
| `instance_ip` | string | Private IP address | Service discovery |
| `kms_id_01` | string | KMS key identifier | Key reference |
| `ami_id` | string | Deployed AMI ID | Audit trail |
| `ami_name` | string | AMI name | Documentation |
| `iam_role_name` | string | IAM role name | Cross-reference |
| `iam_instance_profile_name` | string | Instance profile | Troubleshooting |
| `s3_bucket_name` | string | S3 bucket name | Data access |

## Operations

### Deployment Process

1. **Prerequisites**
```bash
# Verify AWS credentials
aws sts get-caller-identity

# Check VPC and subnet
aws ec2 describe-vpcs --vpc-ids vpc-xxx
aws ec2 describe-subnets --subnet-ids subnet-xxx
```

2. **Terraform Initialization**
```bash
cd cosigner/
terraform init
terraform plan -var-file="terraform.tfvars"
```

3. **Apply Configuration**
```bash
terraform apply -var-file="terraform.tfvars" -auto-approve
```

4. **Verification**
```bash
# Check instance status
aws ec2 describe-instances --instance-ids $(terraform output -raw instance_id)

# Verify enclave status
aws ssm start-session --target $(terraform output -raw instance_id)
nitro-cli describe-enclaves
```

### Monitoring

#### CloudWatch Metrics
- **CPU Utilization**: Monitor enclave CPU usage
- **Network In/Out**: Track API traffic
- **Disk Usage**: Monitor storage consumption
- **Custom Metrics**: Application-specific metrics

#### CloudWatch Logs
```
/aws/ec2/nitro-mainnet-01/
├── system.log
├── application.log
├── enclave.log
└── audit.log
```

#### Alarms
| Metric | Threshold | Action |
|--------|-----------|--------|
| CPU Utilization | > 80% | Scale notification |
| Disk Usage | > 90% | Cleanup alert |
| Failed API Calls | > 10/min | Investigation |
| KMS Throttling | Any | Performance review |

### Maintenance

#### Enclave Updates
1. Build new enclave image
2. Calculate new PCR values
3. Update KMS key policy with new PCR8
4. Deploy new image to instance
5. Verify attestation

#### Security Patches
```bash
# Connect via Session Manager
aws ssm start-session --target instance-id

# Update system packages
sudo yum update -y

# Restart enclave if needed
sudo systemctl restart nitro-enclaves-allocator
```

#### Backup Procedures
1. **Configuration**: Terraform state in version control
2. **Key Shares**: S3 versioning or cross-region replication
3. **Audit Logs**: Archive to S3 Glacier
4. **Attestation Documents**: Retain for compliance

### Troubleshooting

#### Common Issues

1. **Enclave Attestation Failure**
   - Verify PCR8 value in KMS policy
   - Check enclave image integrity
   - Review CloudWatch logs

2. **KMS Access Denied**
   - Verify IAM role permissions
   - Check attestation document
   - Review KMS key policy

3. **S3 Access Issues**
   - Verify bucket policy
   - Check IAM role attachment
   - Review S3 access logs

4. **Network Connectivity**
   - Verify security group rules
   - Check subnet routing
   - Review VPC flow logs

### Security Best Practices

1. **Access Control**
   - Use Session Manager only (no SSH)
   - Implement MFA for administrative access
   - Regular access reviews

2. **Data Protection**
   - Enable S3 versioning
   - Implement backup encryption
   - Regular key rotation

3. **Monitoring**
   - Enable CloudTrail logging
   - Configure GuardDuty
   - Implement custom security metrics

4. **Compliance**
   - Regular security assessments
   - Attestation verification
   - Audit log retention

## Cost Analysis

### Monthly Cost Breakdown
| Component | Specification | Estimated Cost |
|-----------|--------------|----------------|
| EC2 Instance | c5.xlarge (730 hours) | $124.00 |
| EBS Storage | 100 GB GP3 | $8.00 |
| S3 Storage | 10 GB | $0.23 |
| KMS Operations | 10,000 requests | $0.03 |
| Data Transfer | 100 GB | $9.00 |
| **Total** | | **~$141.26** |

### Cost Optimization
- Consider Reserved Instances for 40-60% savings
- Use S3 lifecycle policies for old data
- Monitor and optimize data transfer
- Right-size instance based on actual usage

## Disaster Recovery

### Recovery Objectives
- **RTO**: 30 minutes (instance replacement)
- **RPO**: 5 minutes (S3 replication lag)

### Backup Strategy
1. **Infrastructure**: Terraform in Git
2. **Data**: S3 cross-region replication
3. **Configuration**: Parameter Store backup
4. **Keys**: KMS multi-region keys (optional)

### Recovery Procedures
1. **Instance Failure**: Auto-recovery or manual replacement
2. **Region Failure**: Deploy to alternate region
3. **Data Corruption**: Restore from S3 versions
4. **Key Compromise**: Rotate keys and redeploy

---

*Module Version: 1.0*  
*Last Updated: 2025*  
*Terraform Version: >= 0.12*  
*AWS Provider Version: ~> 5.0*