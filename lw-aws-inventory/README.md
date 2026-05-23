# FortiCNAPP AWS Inventory — Container Image

Runs [`lw_aws_inventory.sh`](lw_aws_inventory.sh) inside Docker.  
**AWS CLI v2** and **jq** are baked into the image — nothing to install on your machine except Docker.

---

## Prerequisites

| Requirement | How to get it |
|-------------|--------------|
| Docker | https://docs.docker.com/get-docker/ |
| AWS credentials | Access key + secret **or** an IAM role |

> No AWS CLI, no jq, no bash — all included in the image.

---

## Step 1 — Clone this repo

```bash
git clone https://github.com/svuillaume/container_images.git
cd container_images/lw-aws-inventory
```

---

## Step 2 — Build the image

```bash
docker build -t lw-aws-inventory .
```

> ⏱ First build takes ~2 minutes — it downloads and installs AWS CLI v2.  
> Subsequent builds use the Docker cache and are instant.  
> Works on both **Intel (amd64)** and **Apple Silicon / AWS Graviton (arm64)**.

---

## Step 3 — Set your AWS credentials

The container needs valid AWS credentials to call AWS APIs.  
Choose **one** of the following methods:

### Option A — Environment variables (most common)

```bash
export AWS_ACCESS_KEY_ID=AKIAxxxxxxxxxxxxxxxx
export AWS_SECRET_ACCESS_KEY=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
export AWS_DEFAULT_REGION=us-east-1
```

### Option B — Named AWS profile (uses your ~/.aws config)

No export needed — just pass your profile name with `-p` in Step 4.

### Option C — IAM role (ECS / EC2 / EKS)

No credentials needed — the container picks them up automatically.

---

## Step 4 — Run the scan

### Single account — all regions

```bash
docker run --rm \
  -e AWS_ACCESS_KEY_ID \
  -e AWS_SECRET_ACCESS_KEY \
  -e AWS_DEFAULT_REGION \
  lw-aws-inventory
```

### Single account — specific regions (faster, recommended)

```bash
docker run --rm \
  -e AWS_ACCESS_KEY_ID \
  -e AWS_SECRET_ACCESS_KEY \
  lw-aws-inventory -r us-east-1,us-west-2,eu-west-1
```

### Named AWS profile

```bash
docker run --rm \
  -v ~/.aws:/home/scanner/.aws:ro \
  lw-aws-inventory -p myprofile -r us-east-1
```

### Entire AWS Organization (all accounts, all OUs)

```bash
docker run --rm \
  -e AWS_ACCESS_KEY_ID \
  -e AWS_SECRET_ACCESS_KEY \
  lw-aws-inventory -o OrganizationAccountAccessRole
```

> Replace `OrganizationAccountAccessRole` with your actual cross-account role name  
> (e.g. `AWSControlTowerExecution` for Control Tower environments).

### One specific account within an organization

```bash
docker run --rm \
  -e AWS_ACCESS_KEY_ID \
  -e AWS_SECRET_ACCESS_KEY \
  lw-aws-inventory -o OrganizationAccountAccessRole -a 123456789012
```

---

## Step 5 — Save the results

### Save CSV to a file

```bash
docker run --rm \
  -e AWS_ACCESS_KEY_ID \
  -e AWS_SECRET_ACCESS_KEY \
  lw-aws-inventory --output csv > results.csv
```

### Summary only (no CSV)

```bash
docker run --rm \
  -e AWS_ACCESS_KEY_ID \
  -e AWS_SECRET_ACCESS_KEY \
  lw-aws-inventory --output summary
```

---

## What the output looks like

```
"Profile","Account ID","Regions","EC2 Instances","EC2 vCPUs","ECS Fargate Clusters",
"ECS Fargate Running Tasks","ECS Fargate CPU Units","ECS Fargate License vCPUs",
"Lambda Functions (not used for licensing)","Total vCPUs"
"","123456789012","us-east-1|us-west-2","42","168","3","12","6144","6","25","174"

######################################################################
  FortiCNAPP inventory collection complete  (87s)
######################################################################

  Accounts Analyzed      : 1

  EC2
  ─────────────────────────────────
  Instances              : 42
  vCPUs                  : 168

  ECS Fargate
  ─────────────────────────────────
  Clusters               : 3
  Running Tasks          : 12
  License vCPUs          : 6

  License Estimate
  ─────────────────────────────────
    EC2 vCPUs            : 168
  + ECS Fargate vCPUs   : 6
  ─────────────────────────────────
  = Total vCPUs           : 174      ← this number goes to FortiCNAPP for sizing
```

---

## All flags

| Flag | Description | Example |
|------|-------------|---------|
| `-r REGIONS` | Limit scan to specific regions | `-r us-east-1,eu-west-1` |
| `-p PROFILE` | Use a named AWS CLI profile | `-p production` |
| `-o ROLE` | Cross-account role for org scanning | `-o OrganizationAccountAccessRole` |
| `-a ACCOUNT_ID` | Scan one account in an org (needs `-o`) | `-a 123456789012` |
| `--output FORMAT` | Output format | `--output csv` |
| `-g FILE` | Generate a per-account script | `-g scan.sh` |
| `-h` | Show help | |

**Output formats:** `all` (default — CSV + summary) · `csv` · `summary` · `csvnoheader`

---

## Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| `Could not resolve account ID` | No AWS credentials passed | Add `-e AWS_ACCESS_KEY_ID -e AWS_SECRET_ACCESS_KEY` |
| `ExpiredToken` | Temporary credentials expired | Re-export fresh credentials and re-run |
| `AccessDenied` on assume-role | Cross-account role not configured | Check the role's trust policy allows your account |
| All counts show 0 | Credentials valid but wrong region | Add `-r` with the regions where your workloads run |
| `ERROR: No access to region X` | Region disabled in your account | Normal — use `-r` to skip disabled regions |

---

## IAM Permissions Required

Minimum policy for the AWS identity running the scan:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeRegions",
        "ec2:DescribeInstances",
        "ecs:ListClusters",
        "ecs:ListTasks",
        "ecs:DescribeTasks",
        "lambda:ListFunctions",
        "sts:GetCallerIdentity"
      ],
      "Resource": "*"
    }
  ]
}
```

For **AWS Organization scanning**, also add:

```json
{
  "Effect": "Allow",
  "Action": [
    "organizations:ListAccounts",
    "sts:AssumeRole"
  ],
  "Resource": "*"
}
```

---

## What's inside the image

| Component | Details |
|-----------|---------|
| Base OS | `debian:bookworm-slim` |
| AWS CLI | v2 — latest at build time |
| jq | latest from apt |
| bash | system |
| User | `scanner` (non-root, UID 1000) |
| Architectures | `amd64` (Intel/AMD) · `arm64` (Apple Silicon, AWS Graviton) |
