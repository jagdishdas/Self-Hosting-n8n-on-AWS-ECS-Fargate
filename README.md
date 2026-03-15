# Self-Hosting n8n on AWS ECS Fargate

> A complete, production-grade deployment of [n8n](https://n8n.io) — the open-source workflow automation tool — on AWS using ECS Fargate, RDS PostgreSQL, EFS, ALB, and Secrets Manager. Built entirely from the AWS CLI on Windows 11.

---

## Architecture

```
Internet
    │
    ▼
Application Load Balancer  (Public Subnet — us-east-1a / 1b)
    │  port 80
    ▼
ECS Fargate Task  (Private Subnet — us-east-1a)
    │  n8n container  port 5678
    ├──► RDS PostgreSQL  (Private Subnet — workflow & credential storage)
    ├──► EFS File System  (Persistent config at /home/node/.n8n)
    └──► Secrets Manager  (DB password + encryption key injected at runtime)
         CloudWatch Logs  (/ecs/n8n log group)
```

**Network layout:**
- `10.0.0.0/16` — VPC
- `10.0.1.0/24` — Public Subnet A (ALB)
- `10.0.2.0/24` — Public Subnet B (ALB — 2nd AZ required)
- `10.0.3.0/24` — Private Subnet (ECS Task + RDS)

---

## AWS Services Used

| Service | Purpose |
|---|---|
| **ECS Fargate** | Runs the n8n Docker container — serverless, no EC2 |
| **ECR** | Private Docker image registry — ECS pulls from here |
| **RDS PostgreSQL** | Managed database — durable workflow and credential storage |
| **EFS** | Persistent file system — mounts at `/home/node/.n8n` |
| **ALB** | Application Load Balancer — public entry point, health checks |
| **Secrets Manager** | Stores DB password and n8n encryption key securely |
| **VPC + Subnets** | Network isolation — public for ALB, private for app and DB |
| **NAT Gateway** | Allows private subnet to reach ECR and Secrets Manager APIs |
| **IAM Roles** | Execution Role (pull image, write logs) + Task Role (EFS access) |
| **CloudWatch Logs** | Container log streaming — log group `/ecs/n8n` |
| **EFS Access Points** | Sets correct UID/GID (1000) so n8n can write to EFS |

---

## Prerequisites

- AWS account with IAM user that has AdministratorAccess (for learning)
- AWS CLI v2 installed and configured (`aws configure`)
- Docker Desktop installed and running
- PowerShell 5+ (Windows) or Bash (Mac/Linux)

---

## Deployment Steps

### 1. Push n8n Image to ECR

```powershell
$REGION = "us-east-1"
$ACCOUNT_ID = (aws sts get-caller-identity --query Account --output text)

# Create ECR repo
aws ecr create-repository --repository-name n8n --region $REGION

# Pull from n8n's registry, tag for ECR, push
docker pull docker.n8n.io/n8nio/n8n:latest
docker tag docker.n8n.io/n8nio/n8n:latest "$ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/n8n:latest"

aws ecr get-login-password --region $REGION | `
  docker login --username AWS --password-stdin "$ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com"

docker push "$ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/n8n:latest"
```

### 2. Create VPC and Networking

```powershell
# VPC
$VPC_ID = (aws ec2 create-vpc --cidr-block 10.0.0.0/16 --query 'Vpc.VpcId' --output text)
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-hostnames

# Subnets
$PUB_SUBNET  = (aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.1.0/24 --availability-zone ${REGION}a --query 'Subnet.SubnetId' --output text)
$PUB_SUBNET2 = (aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.2.0/24 --availability-zone ${REGION}b --query 'Subnet.SubnetId' --output text)
$PRIV_SUBNET = (aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.3.0/24 --availability-zone ${REGION}a --query 'Subnet.SubnetId' --output text)

# Internet Gateway for public subnets
$IGW_ID = (aws ec2 create-internet-gateway --query 'InternetGateway.InternetGatewayId' --output text)
aws ec2 attach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID

# NAT Gateway for private subnet (REQUIRED — without this ECS cannot reach ECR)
$EIP_ALLOC = (aws ec2 allocate-address --domain vpc --query 'AllocationId' --output text)
$NAT_ID = (aws ec2 create-nat-gateway --subnet-id $PUB_SUBNET --allocation-id $EIP_ALLOC --query 'NatGateway.NatGatewayId' --output text)
```

### 3. Create Security Groups

Three security groups with strict ingress rules:

- **ALB SG** — accepts 80/443 from `0.0.0.0/0`
- **ECS SG** — accepts 5678 only from ALB SG + 2049 (NFS) from itself
- **RDS SG** — accepts 5432 only from ECS SG

### 4. Create RDS PostgreSQL

```powershell
aws rds create-db-instance `
  --db-instance-identifier n8n-postgres `
  --db-instance-class db.t3.micro `
  --engine postgres `
  --master-username n8nuser `
  --master-user-password "YourPassword" `
  --db-name n8n `
  --vpc-security-group-ids $RDS_SG `
  --db-subnet-group-name n8n-db-subnet `
  --no-publicly-accessible `
  --allocated-storage 20
```

### 5. Store Secrets

```powershell
aws secretsmanager create-secret --name "n8n/db-password" --secret-string "YourPassword"

$bytes = New-Object Byte[] 32
[System.Security.Cryptography.RandomNumberGenerator]::Create().GetBytes($bytes)
$encKey = [BitConverter]::ToString($bytes).Replace("-","").ToLower()
aws secretsmanager create-secret --name "n8n/encryption-key" --secret-string $encKey
```

### 6. Create EFS with Access Point

```powershell
$EFS_ID = (aws efs create-file-system --performance-mode generalPurpose --query 'FileSystemId' --output text)

# Access Point sets UID=1000 (node user) so n8n can write files
aws efs create-access-point `
  --file-system-id $EFS_ID `
  --posix-user "Uid=1000,Gid=1000" `
  --root-directory "Path=/n8n,CreationInfo={OwnerUid=1000,OwnerGid=1000,Permissions=755}"
```

### 7. Create IAM Roles

Two roles required:

**Execution Role** (`n8nECSExecutionRole`) — used by ECS to launch the task:
- `AmazonECSTaskExecutionRolePolicy` (managed)
- Inline policy: `secretsmanager:GetSecretValue` on `n8n/*` secrets

**Task Role** (`n8nECSTaskRole`) — used by the running container:
- Inline policy: `elasticfilesystem:ClientMount` and `ClientWrite` on your EFS

### 8. Register Task Definition

Key environment variables in the task definition:

```json
{ "name": "DB_TYPE",                             "value": "postgresdb" },
{ "name": "DB_POSTGRESDB_SSL_ENABLED",           "value": "true" },
{ "name": "DB_POSTGRESDB_SSL_REJECT_UNAUTHORIZED","value": "false" },
{ "name": "N8N_SECURE_COOKIE",                   "value": "false" },
{ "name": "N8N_PROTOCOL",                        "value": "http" }
```

> **Important:** Save the task definition JSON using `[System.IO.File]::WriteAllText()` on Windows — not `Out-File`. PowerShell's `Out-File -Encoding utf8` adds a BOM that AWS CLI cannot parse.

### 9. Create ALB + Target Group

```powershell
aws elbv2 create-load-balancer --name n8n-alb --subnets $PUB_SUBNET $PUB_SUBNET2 `
  --security-groups $ALB_SG --scheme internet-facing --type application

aws elbv2 create-target-group --name n8n-tg --protocol HTTP --port 5678 `
  --vpc-id $VPC_ID --target-type ip --health-check-path "/healthz"
```

### 10. Create ECS Service

```powershell
aws ecs create-service `
  --cluster n8n-cluster `
  --service-name n8n-service `
  --task-definition n8n-task:1 `
  --desired-count 1 `
  --launch-type FARGATE `
  --network-configuration "awsvpcConfiguration={subnets=[$PRIV_SUBNET],securityGroups=[$ECS_SG],assignPublicIp=DISABLED}" `
  --load-balancers "targetGroupArn=$TG_ARN,containerName=n8n,containerPort=5678" `
  --health-check-grace-period-seconds 60
```

---

## Updating n8n

```powershell
docker pull docker.n8n.io/n8nio/n8n:latest
docker tag docker.n8n.io/n8nio/n8n:latest "$ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/n8n:latest"
docker push "$ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/n8n:latest"

aws ecs update-service --cluster n8n-cluster --service n8n-service --force-new-deployment
```

---

## Monitoring & Debugging

```powershell
# Live logs
aws logs tail /ecs/n8n --follow

# Check service health
aws ecs describe-services --cluster n8n-cluster --services n8n-service `
  --query 'services[0].{Running:runningCount,Desired:desiredCount,Event:events[0].message}'

# Check ALB target health
aws elbv2 describe-target-health --target-group-arn $TG_ARN `
  --query 'TargetHealthDescriptions[].{IP:Target.Id,State:TargetHealth.State}'
```

---

## Common Errors and Fixes

| Error | Root Cause | Fix |
|---|---|---|
| `ResourceInitializationError: EFS mount timeout` | ECS in private subnet, no NAT Gateway | Add NAT Gateway + private route table |
| `ECS unable to assume role` | IAM role trust policy missing `ecs-tasks.amazonaws.com` | Recreate role with correct trust policy |
| `no pg_hba.conf entry...no encryption` | RDS requires SSL | Add `DB_POSTGRESDB_SSL_ENABLED=true` |
| `EACCES: permission denied /home/node/.n8n` | EFS mounts as root, n8n runs as UID 1000 | Add EFS Access Point with `Uid=1000,Gid=1000` |
| `Invalid JSON received` | PowerShell BOM in UTF-8 file | Use `[System.IO.File]::WriteAllText()` |
| `secure cookie / insecure URL` | n8n requires HTTPS for cookies | Set `N8N_SECURE_COOKIE=false` for HTTP |

---

## Cost Estimate

| Resource | ~Monthly Cost |
|---|---|
| ECS Fargate (0.5 vCPU, 1 GB, 24/7) | $15 |
| RDS PostgreSQL (db.t3.micro) | $15 |
| ALB | $16 |
| NAT Gateway | $32 |
| EFS | $1 |
| **Total** | **~$79/month** |

> Stop RDS and set `desired-count=0` when not actively using to reduce cost during learning.

---

## References

- [n8n Self-Hosting Docs](https://docs.n8n.io/hosting/)
- [AWS ECS Fargate Documentation](https://docs.aws.amazon.com/ecs/)
- [AWS EFS with ECS](https://docs.aws.amazon.com/ecs/latest/bestpracticesguide/storage-efs.html)

---

*Built as part of an AWS cloud learning journey. Every error documented, every fix applied.*
