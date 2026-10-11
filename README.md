# two-tier-app

A small Python app (nginx frontend + Flask backend) deployed to Amazon EKS.
Infrastructure lives in the separate **two-tier-infra** repo.

```
backend/            Flask API, tests, Dockerfile
frontend/           index.html, nginx.conf, Dockerfile
k8s/                Deployments and Services for both tiers
.github/workflows/  ci.yml (PRs/branches), deploy.yml (main -> EKS)
docker-compose.yml  run both tiers locally
```

**Frontend tier:** nginx serves the page and proxies `/api/*` to the backend. It's exposed through a LoadBalancer.
**Backend tier:** Flask REST API (`/api/health`, `/api/info`, `/api/tasks`). It's internal only (ClusterIP).

## Run locally (free)

```bash
docker compose up --build        # http://localhost:8080
```

Or on a local Kubernetes cluster with kind:

```bash
kind create cluster --name two-tier
docker build -t backend:local ./backend && docker build -t frontend:local ./frontend
kind load docker-image backend:local frontend:local --name two-tier
kubectl create namespace two-tier
sed "s|BACKEND_IMAGE|backend:local|" k8s/backend.yaml | kubectl apply -f -
sed "s|FRONTEND_IMAGE|frontend:local|" k8s/frontend.yaml | kubectl apply -f -
kubectl port-forward -n two-tier svc/frontend 8080:80
```

## Deploy to EKS

1. Bring the cluster up from the infra repo (Actions > Terraform > Run workflow).
2. The first time only: copy the values from that run's job summary into this repo's Settings > Secrets and variables > Actions.

| Type | Name |
|---|---|
| Secret | `AWS_ROLE_ARN` |
| Variable | `AWS_REGION`, `EKS_CLUSTER`, `ECR_BACKEND`, `ECR_FRONTEND` |

3. Push to `main`, or go to Actions > Build and deploy to EKS > Run workflow.
   - The pipeline runs the tests, builds and pushes both images tagged with the commit SHA, deploys the backend and then the frontend, and prints the app URL.

The values never change between infra rebuilds, so after a destroy and recreate you just re-run the deploy workflow.

## What this pipeline is allowed to do

Its IAM role can push to the two ECR repos and has edit rights in the `two-tier` namespace only. It cannot create namespaces, touch other workloads, or change infrastructure. That's the point of the split.

## Note

Tasks are stored in backend memory, so the backend runs 1 replica. Add Redis or RDS to make it 3-tier.
