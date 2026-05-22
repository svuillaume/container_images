# lw-aws-inventory container

A minimal Docker image that runs [`lw_aws_inventory.sh`](lw_aws_inventory.sh) — the
FortiCNAPP AWS license vCPU sizing script — with **AWS CLI v2** and **jq** baked in.

No local install of AWS CLI or jq required. Just Docker + AWS credentials.

---

## What's inside the image

| Component | Version | Purpose |
|-----------|---------|---------|
| debian:bookworm-slim | latest | Base OS |
| AWS CLI | v2 (latest) | Required by the script |
| jq | latest | Required by the script |
| bash | system | Required to run the script |
| lw_aws_inventory.sh | latest | The sizing script |

> Runs as a non-root user (`scanner`, UID 1000).  
> Supports both `amd64` and `arm64` (Apple Silicon / AWS Graviton).

---

## ⚠️ AWS Credentials Required

The container calls live AWS APIs. Pass credentials via environment variables:

```bash
-e AWS_ACCESS_KEY_ID=AKIAxxxxxxxxxxxxxxxxx
-e AWS_SECRET_ACCESS_KEY=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
-e AWS_DEFAULT_REGION=us-east-1
```

Or mount your `~/.aws` directory (for named profiles):

```bash
-v ~/.aws:/home/scanner/.aws:ro
```

Or use an **IAM Instance Role / ECS Task Role / IRSA** — credentials are picked up
automatically with no extra flags.

---

## Quick Start

### Build locally

```bash
cd lw-aws-inventory
docker build -t lw-aws-inventory .
```

### Run — default credentials, all regions

```bash
docker run --rm \
  -e AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID \
  -e AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY \
  -e AWS_DEFAULT_REGION=us-east-1 \
  lw-aws-inventory
```

### Run — specific regions (faster)

```bash
docker run --rm \
  -e AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID \
  -e AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY \
  lw-aws-inventory -r us-east-1,us-west-2,eu-west-1
```

### Run — named profile (mount ~/.aws)

```bash
docker run --rm \
  -v ~/.aws:/home/scanner/.aws:ro \
  lw-aws-inventory -p myprofile -r us-east-1
```

### Run — entire AWS Organization

```bash
docker run --rm \
  -e AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID \
  -e AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY \
  lw-aws-inventory -o OrganizationAccountAccessRole
```

### Save CSV output to a file

```bash
docker run --rm \
  -e AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID \
  -e AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY \
  lw-aws-inventory --output csv > results.csv
```

### Show help

```bash
docker run --rm lw-aws-inventory --help
```

---

## All flags

| Flag | Description |
|------|-------------|
| `-r REGIONS` | Comma-separated regions, e.g. `us-east-1,us-west-2` |
| `-p PROFILE` | AWS CLI profile name |
| `-o ROLE` | Cross-account role for AWS Organization scan |
| `-a ACCOUNT_ID` | Scan one account within an org (requires `-o`) |
| `-g FILE` | Generate a per-account script instead of running |
| `--output FORMAT` | `all` (default) \| `csv` \| `summary` \| `csvnoheader` |
| `-h` | Show help |

---

## Environment variables

| Variable | Required | Description |
|----------|----------|-------------|
| `AWS_ACCESS_KEY_ID` | Yes (or role) | AWS access key |
| `AWS_SECRET_ACCESS_KEY` | Yes (or role) | AWS secret key |
| `AWS_SESSION_TOKEN` | If using STS | Temporary session token |
| `AWS_DEFAULT_REGION` | Recommended | Default region for API calls |

---

## IAM Permissions Required

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

Add `organizations:ListAccounts` and `sts:AssumeRole` for organization scanning.
