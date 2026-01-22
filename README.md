# AI Digital Twin (AWS Bedrock + Lambda + API Gateway + S3 + CloudFront)

This project deploys a static Next.js chat UI to **S3 + CloudFront** and a FastAPI backend to **AWS Lambda** (behind **API Gateway**). Chat responses come from **AWS Bedrock** (Nova models by default). Conversation history is persisted in **S3**.

## Repo layout

- `backend/`: FastAPI app, Lambda handler, packaging script, and personalization data
- `frontend/`: Next.js UI (static export)
- `terraform/`: Infrastructure as code (Lambda, API Gateway, S3, CloudFront, optional custom domain)
- `scripts/`: Convenience scripts to deploy/destroy with Terraform

## Prerequisites

- **AWS**: configured credentials locally (for local deploy) and **Bedrock model access enabled** in your AWS account/region
- **Node.js**: 18+ (Node 20 recommended)
- **Python**: 3.11+ (3.12 recommended)
- **AWS CLI**: v2
- **Docker**: required to build the Lambda zip consistently
- **Terraform**: required for local `scripts/deploy.sh` / `scripts/destroy.sh`

## Run locally

### Backend (FastAPI)

```bash
cd backend
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
uvicorn server:app --reload --port 8000
```

Notes:
- The backend calls **Bedrock**, so your local environment must have AWS credentials available (e.g. `AWS_PROFILE`, `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY`) and `DEFAULT_AWS_REGION` set if needed.
- Local memory defaults to `backend/../memory` unless you set `USE_S3=true`.

### Frontend (Next.js dev)

```bash
cd frontend
npm install
echo "NEXT_PUBLIC_API_URL=http://localhost:8000" > .env.local
npm run dev
```

Open `http://localhost:3000`.

## Deploy to AWS (Terraform)

### 1) One-time: bootstrap Terraform backend (state + locking)

The deploy scripts configure Terraform to use:
- S3 bucket: `twin-terraform-state-$AWS_ACCOUNT_ID`
- DynamoDB table: `twin-terraform-locks`

Create them once (adjust region if needed):

```bash
export DEFAULT_AWS_REGION=us-east-1
export AWS_ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"

aws s3 mb "s3://twin-terraform-state-${AWS_ACCOUNT_ID}" --region "$DEFAULT_AWS_REGION"

aws dynamodb create-table \
  --region "$DEFAULT_AWS_REGION" \
  --table-name twin-terraform-locks \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
```

### 2) Deploy

```bash
export DEFAULT_AWS_REGION=us-east-1
chmod +x scripts/deploy.sh
./scripts/deploy.sh dev
```

This will:
- build `backend/lambda-deployment.zip`
- `terraform apply` to create/update infra
- build the frontend static export and upload to the S3 frontend bucket

### 3) Destroy

```bash
chmod +x scripts/destroy.sh
./scripts/destroy.sh dev
```

## GitHub Actions (optional)

Workflows:
- `/.github/workflows/deploy.yml`: deploys on push to `main` (or manual dispatch)
- `/.github/workflows/destroy.yml`: destroys an environment (manual dispatch)

You’ll need to configure repository secrets (and an AWS role for GitHub OIDC):
- `AWS_ROLE_ARN`
- `AWS_ACCOUNT_ID`
- `DEFAULT_AWS_REGION`

## Troubleshooting

### UI shows “Sorry, I encountered an error…”

That message is shown when the frontend’s `/chat` request fails.

- **Check browser DevTools → Network**: look for the failing `POST /chat` and its status code.
- **Check Lambda logs**: CloudWatch log group should be `/aws/lambda/<function-name>`.

Common causes:
- **Bedrock access denied / model not enabled**: enable model access in Bedrock, and ensure Lambda role has Bedrock permissions.
- **Wrong Bedrock model id for your region**: try `us.amazon.nova-lite-v1:0` or `eu.amazon.nova-lite-v1:0` (depending on your region).
- **CORS mismatch**: `CORS_ORIGINS` must match your CloudFront domain exactly (include `https://`, no trailing slash).

## Security / Git hygiene

- Never commit secrets. `.env*` is ignored by `./.gitignore`.
- Terraform state (`*.tfstate*`) is ignored.
- Lambda build artifacts (`backend/lambda-deployment.zip`, `backend/lambda-package/`) are ignored.

If you plan to make the repository **public**, review/remove/replace any personal content in:
- `backend/data/*` (e.g. `linkedin.pdf`, `facts.json`, etc.)
- `backend/me.txt`


