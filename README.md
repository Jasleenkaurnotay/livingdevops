# devops-games on AWS ECS

A gamified DevOps learning platform built with Flask, deployed on AWS ECS Fargate.
Players complete interactive mini-games (Dockerfile builder, K8s YAML fixer, incident
commander, etc.), earn scores, unlock badges, and compete on a leaderboard.

---

## Application Architecture
Internet → ALB (port 80) → Nginx (port 80) → Flask/Gunicorn (port 8080)

↓              ↓

Redis           RDS

(port 6379)   Postgres

Celery broker   (port 5432)

### Services
| Service | Role                                         | Port |
|---------|----------------------------------------------|------|
| Nginx   | Reverse proxy — entry point for all traffic  | 80   |
| Flask   | Backend — serves app, APIs, and static files | 8080 |
| Redis   | Cache + Celery task broker                   | 6379 |
| RDS Postgres | Persistent storage for scores, profiles, achievements | 5432 |

> Note: Nginx is a pure reverse proxy here — it does not serve static files directly.
> All static assets (CSS, JS) are served by Flask itself.

> Note: Celery worker is not deployed in this version. The async email demo page
> (/celery) will not function. All game features work normally.

---

## Tech Stack

- **App**: Python 3.11, Flask, Gunicorn, Celery, SQLAlchemy
- **Database**: PostgreSQL 14 (RDS — single instance on dev, Aurora cluster on prod)
- **Cache/Queue**: Redis
- **Proxy**: Nginx
- **Container runtime**: Docker
- **Orchestration**: AWS ECS Fargate
- **Infrastructure as Code**: Terraform (modularized)
- **Image registry**: AWS ECR
- **Networking**: AWS VPC, ALB, Route53, NAT Gateway
- **Secrets**: AWS Secrets Manager
- **Logging**: AWS CloudWatch

---

## Repo Structure
.

├── app/                        # Application code

│   ├── app.py                  # Flask app — routes, Celery tasks

│   ├── config.py               # App config (reads from env vars)

│   ├── models.py               # SQLAlchemy models

│   ├── games_data.py           # Game content and scenarios

│   ├── requirements.txt        # Python dependencies

│   ├── run.sh                  # Gunicorn entrypoint

│   ├── Dockerfile              # Flask app image

│   ├── docker-compose.yml      # Local development setup

│   ├── nginx/

│   │   ├── Dockerfile          # Nginx image

│   │   ├── nginx.conf          # Production nginx config

│   │   └── nginx-local.conf    # Local docker compose nginx config

│   ├── redis/

│   │   └── Dockerfile          # Redis image (extends official image)

│   ├── static/                 # CSS and JS assets

│   └── templates/              # Jinja2 HTML templates

│       └── games/              # Individual game templates

└── infra/                      # Terraform infrastructure code

├── main.tf                 # Root module — wires all modules together

├── locals.tf               # SG assignment logic per service

├── variables.tf            # Root variable declarations

├── outputs.tf              # Root outputs

├── providers.tf            # AWS provider + S3 backend config

├── oidc.tf                 # OIDC config (GitHub Actions ready)

├── modules/

│   ├── network/            # VPC, subnets, IGW, NAT, security groups

│   ├── ecs/                # ECS cluster, task defs, services, ALB

│   └── database/           # RDS instance (dev) / Aurora cluster (prod)

└── vars/

├── dev.tfvars          # Dev environment variable values

├── prod.tfvars         # Prod environment variable values

├── dev.tfbackend       # Dev S3 backend config

└── prod.tfbackend      # Prod S3 backend config

---

## Running Locally

```bash
cd app
docker compose up --build
```

Access the app at `http://localhost:8000`

The local setup uses:
- Flask app container (port 8080 internally)
- Nginx as reverse proxy (exposed on port 8000)
- Redis container
- Postgres container (no RDS needed locally)

---

## Deploying to AWS

### Prerequisites
- AWS CLI configured with appropriate profile (`terraform`)
- Terraform >= 1.0
- Docker (with buildx for multi-platform builds)
- S3 bucket for Terraform state (`mylabs-terraform-state`)

### Step 1 — Create ECR Repositories
Create 3 repositories manually in the AWS Console:
- `devops-games/app`
- `devops-games/nginx`
- `devops-games/redis`

### Step 2 — Build and Push Images

```bash
# ECR login
aws ecr get-login-password --region us-east-1 | docker login --username AWS \
  --password-stdin 067270456427.dkr.ecr.us-east-1.amazonaws.com

# Build (--platform linux/amd64 required for Fargate if building on Apple Silicon)
docker build --platform linux/amd64 -t devops-games-app:latest ./app
docker build --platform linux/amd64 -t devops-games-nginx:latest ./app/nginx
docker build --platform linux/amd64 -t devops-games-redis:latest ./app/redis

# Tag
docker tag devops-games-app:latest   067270456427.dkr.ecr.us-east-1.amazonaws.com/devops-games/app:latest
docker tag devops-games-nginx:latest 067270456427.dkr.ecr.us-east-1.amazonaws.com/devops-games/nginx:latest
docker tag devops-games-redis:latest 067270456427.dkr.ecr.us-east-1.amazonaws.com/devops-games/redis:latest

# Push
docker push 067270456427.dkr.ecr.us-east-1.amazonaws.com/devops-games/app:latest
docker push 067270456427.dkr.ecr.us-east-1.amazonaws.com/devops-games/nginx:latest
docker push 067270456427.dkr.ecr.us-east-1.amazonaws.com/devops-games/redis:latest
```

### Step 3 — Create Secrets Manager Secret
Before running Terraform, create the DB password secret:
```bash
aws secretsmanager create-secret \
  --name devops-games-dev-db-url \
  --secret-string "changeme123" \
  --region us-east-1
```

### Step 4 — Run Terraform

```bash
cd infra
terraform init -backend-config=vars/dev.tfbackend
terraform plan -var-file=vars/dev.tfvars
terraform apply -var-file=vars/dev.tfvars
```

### Step 5 — Update RDS Endpoint
After apply, get the RDS endpoint from the AWS Console and update
`DB_ADDRESS` in `vars/dev.tfvars`, then run `terraform apply` again.

---

## Infrastructure Overview

### Networking
- 1 VPC (`192.168.0.0/16`)
- 2 public subnets (ALB lives here)
- 2 private subnets (ECS tasks + RDS live here)
- 1 NAT Gateway (single, shared — cost saving for dev)
- Internet Gateway

### Security Groups (traffic flow)
Internet → ALB sg (80/443) → nginx sg (80) → flask sg (8080) → redis sg (6379)
                                                     ↓
                                               db sg (5432)

### ECS
- 1 ECS Cluster (Fargate)
- 3 ECS Services: nginx, app, redis
- Services communicate via ECS Service Connect (internal DNS)
- Only nginx is behind the ALB

### Database
- Dev: single RDS Postgres instance (`db.t3.micro`)
- Prod: Aurora PostgreSQL cluster (writer + reader)

---

## Known Limitations / Future Work
- Celery worker not deployed — async email demo not functional
- No HTTPS — ALB listener is HTTP only (add ACM cert + Route53 for prod)
- DB password in tfvars — move to Secrets Manager reference for prod
- No autoscaling configured on ECS services