# Multi-Cloud Infrastructure & Application Platform

Welcome to our cloud engineering organization. This README provides an overview of our multi-cloud platform built on **AWS** and **GCP**, covering the deployed services, infrastructure configuration, application stack, and CI/CD pipeline managed with Terraform, Packer, and GitHub Actions.

---

## Repositories

| Repository | Description |
|------------|-------------|
| `webapp` | FastAPI REST API — application core, Alembic migrations, tests, and Packer image template |
| `tf-infra` | Terraform modules for AWS and GCP infrastructure across `dev` and `demo` environments |
| `serverless` | SNS-triggered Lambda function for email verification via Mailgun |

---

## Services Activated

### AWS

- **VPC** with public and private subnets across multiple Availability Zones
- **Application Load Balancer** with HTTPS termination (ACM certificate) and HTTP → HTTPS redirect
- **Auto Scaling Group** with CPU-based scaling and rolling instance refresh on deploy
- **EC2** (Ubuntu 24.04) running the FastAPI application via `systemd`
- **RDS MySQL 8.4** in private subnets with KMS-encrypted storage
- **S3** private bucket with SSE-KMS and lifecycle tiering to STANDARD_IA
- **KMS** — four Customer Managed Keys (EC2, RDS, S3, Secrets Manager) with 90-day rotation
- **Secrets Manager** — DB credentials and Mailgun API key, fetched at runtime
- **SNS** — user registration events published by the API
- **Lambda** (Python 3.12, VPC-attached) — email verification handler
- **IAM** — least-privilege roles for EC2 and Lambda
- **Route 53** — apex domain alias record pointing to the ALB
- **CloudWatch** — agent-based metrics and automatic dashboard provisioning on first boot

### GCP

- **VPC** with custom subnets and subnet flow logs
- **Cloud Router + Cloud NAT** for private subnet egress
- **Firewall** rules (SSH, HTTP, HTTPS, application port, internal, deny-all)
- **Compute Engine** instance provisioned from a custom Packer-built image

---

## Current Configuration

### AWS

| Parameter | Value |
|-----------|-------|
| Region | `us-east-1` |
| VPC CIDR (dev) | `10.0.0.0/16` |
| VPC CIDR (demo) | `10.1.0.0/16` |
| Application port | `80` (EC2) — HTTPS enforced at ALB |
| Database engine | MySQL `8.4.8` |
| EC2 instance type | `t2.micro` (default) |
| DB instance class | `db.t3.micro` (default) |
| ASG min / max | `1 / 2` |
| Health check path | `/healthz` |
| KMS key rotation | 90 days |
| S3 lifecycle | STANDARD → STANDARD_IA after 30 days |

### GCP

| Parameter | Value |
|-----------|-------|
| Region | `us-east1` |
| Zone | `us-east5-c` (Packer default) |
| Image base | Ubuntu 24.04 LTS |

---

## Application Stack

The REST API (`webapp`) is built with:

- **FastAPI** — async Python web framework with auto-generated OpenAPI docs
- **SQLModel + Alembic** — typed ORM with automatic migrations on startup
- **MySQL 8** — relational data store (RDS on AWS)
- **bcrypt** — password hashing with strength validation
- **HTTP Basic Auth** — email-based authentication
- **StatsD** — per-endpoint call count and latency metrics
- **Structured JSON logging** — with `X-Request-ID` request tracing
- **Pydantic Settings** — environment-based configuration

### API Endpoints

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `GET` | `/healthz` | None | Health check with DB verification |
| `POST` | `/v1/user` | None | Create user account |
| `GET` | `/v1/user/self` | Basic | Get authenticated user |
| `PUT` | `/v1/user/self` | Basic | Update authenticated user |
| `GET` | `/v1/user/verify` | None | Email verification |
| `GET` | `/v1/courses` | Basic | List all courses |
| `POST` | `/v1/courses` | Basic | Create course |
| `PUT` | `/v1/courses/{id}` | Basic | Update course |
| `DELETE` | `/v1/courses/{id}` | Basic | Delete course |
| `POST` | `/v1/courses/{id}/syllabus` | Basic | Upload syllabus to S3 |
| `DELETE` | `/v1/courses/{id}/syllabus` | Basic | Delete syllabus from S3 |

---

## Architecture

```
Internet
   │ HTTPS (443)
   ▼
Application Load Balancer  ←  ACM certificate, HTTP→HTTPS redirect
   │ HTTP (app_port)
   ▼
EC2 Auto Scaling Group  ←  Custom Ubuntu AMI (Packer), KMS-encrypted EBS
   │              │              │
   ▼              ▼              ▼
RDS MySQL     S3 Bucket      SNS Topic
(private)    (private,           │
              SSE-KMS)           ▼
                            Lambda Function  ←  VPC-attached, Python 3.12
                                │
                                ▼
                          Mailgun API  →  Verification email to user

All secrets:   AWS Secrets Manager  (KMS-encrypted, fetched at runtime)
All storage:   Customer Managed KMS Keys  (90-day automatic rotation)
DNS:           Route 53 alias  →  ALB
```

---

## Managing Infrastructure with Terraform

### Prerequisites

- Terraform >= 1.0
- AWS CLI configured with named profiles (`dev`, `demo`)
- Google Cloud SDK authenticated via `gcloud auth application-default login`
- ACM certificate issued and validated before first apply

### AWS

```bash
cd tf-infra/aws/environments/dev   # or demo

cp terraform.tfvars.example terraform.tfvars
# Set: aws_profile, vpc_cidr, custom_ami_id, domain_name, acm_certificate_arn, mailgun credentials

terraform init
terraform validate
terraform plan
terraform apply
```

### GCP

```bash
cd tf-infra/gcp/environments/dev   # or demo

cp terraform.tfvars.example terraform.tfvars
# Set: gcp_project, instance_image, domain_name

terraform init
terraform validate
terraform plan
terraform apply
```

### Teardown

```bash
terraform destroy
```

---

## CI/CD Pipeline

### `webapp` — Integration Tests (`ci.yaml`)

Runs on every push and pull request to `main`:

1. Start MySQL service container
2. Install Python 3.14 + `uv`
3. Lint with `ruff`
4. Run Alembic migrations
5. Run `pytest` with coverage
6. Validate Packer template (fmt + validate)

### `webapp` — Deploy Pipeline (`build.yaml`)

Triggers automatically after integration tests pass on `main` (i.e., every PR merge):

```
validate-packer → build-ami (DEV account) → deploy-demo
```

1. **validate-packer** — Packer fmt check and validate
2. **build-ami** — Packer builds AMI in DEV, shared with DEMO account and GCP demo project
3. **deploy-demo** — creates new Launch Template version, updates ASG, starts instance refresh (`MinHealthyPercentage: 50`), polls until `Successful`

### `tf-infra` — Lint (`terraform-lint.yaml`)

Runs on every PR:

- `terraform fmt -check -recursive`
- `terraform validate` (all environments)

### `serverless` — Lambda Deploy (`ci.yaml`)

Packages the Lambda handler and dependencies into a zip, then updates the function code in AWS on every push to `main`.

---

## Security Highlights

- **HTTPS only**: ALB enforces HTTPS; HTTP redirects with 301
- **EC2 isolation**: Application port accessible only from the ALB security group — not from the internet
- **KMS encryption**: EBS, RDS, S3, and Secrets Manager all use dedicated Customer Managed Keys with 90-day rotation
- **Secrets at runtime**: DB password and Mailgun credentials fetched from Secrets Manager at instance launch and Lambda cold-start — never stored in AMIs, user data, or Terraform state
- **Least-privilege IAM**: EC2 and Lambda roles scoped to exact resources they need
- **bcrypt passwords**: User passwords hashed with bcrypt before storage; never returned in API responses
- **VPC-attached Lambda**: Lambda runs inside the VPC to access RDS privately; outbound HTTPS allowed for Mailgun

---

## Key Takeaways and Improvements

- **Multi-cloud image builds**: Packer builds both AWS AMIs and GCP Compute images in a single pipeline, keeping base images consistent across clouds
- **Zero-downtime deploys**: Rolling instance refresh with `MinHealthyPercentage: 50` ensures the DEMO environment stays available during every release
- **Secrets never touch state**: Auto-generated RDS passwords flow directly from `random_password` into Secrets Manager — Terraform state never holds a plaintext credential
- **Four-key KMS architecture**: Separate CMKs per service (EC2, RDS, S3, Secrets Manager) limit blast radius and allow independent rotation or revocation
- **Runtime secret injection**: EC2 `user_data.sh` writes `.env.db` at launch by calling Secrets Manager — no credentials baked into the AMI
- **Network segmentation**: ALB → App → DB security group chain enforces a strict traffic path; Lambda gets its own security group restricted to outbound only
- **Automated database migrations**: Alembic runs `upgrade head` on FastAPI startup — no manual migration step on deploy
- **StatsD + CloudWatch**: Per-endpoint metrics stream to CloudWatch Agent with an auto-deployed dashboard on first boot, giving immediate operational visibility

---

For further details, see the individual repository READMEs, or the full API reference at `webapp/docs/API.md`. Happy deploying!
