# My App — React + Express, Dockerized, deployed on AWS ECS behind an ALB

## Structure
```
myapp/
  frontend/   React (Vite) app, served by Nginx in production
  backend/    Express API
  docker-compose.yml   for local testing of both containers together
```

## 1. Run locally

```bash
docker compose up --build
```
- Frontend: http://localhost:8080
- Backend:  http://localhost:4000/api/hello

The frontend container's Nginx proxies `/api/*` to the backend container
for local testing only. In production on AWS, the ALB does this routing
instead (see below).

---

## 2. Push images to Amazon ECR (AWS Console)

For each service (`frontend`, `backend`):

1. ECR → **Create repository** → name it `myapp-frontend` (repeat for `myapp-backend`).
2. Click the repo → **View push commands** and run them, e.g.:
   ```bash
   aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <account-id>.dkr.ecr.<region>.amazonaws.com

   cd backend
   docker build -t myapp-backend .
   docker tag myapp-backend:latest <account-id>.dkr.ecr.<region>.amazonaws.com/myapp-backend:latest
   docker push <account-id>.dkr.ecr.<region>.amazonaws.com/myapp-backend:latest
   ```
   Repeat for `frontend`.

---

## 3. Create the ECS Cluster

1. ECS → **Create cluster** → Fargate (serverless, no EC2 to manage) → name it `myapp-cluster`.

---

## 4. Create Task Definitions

Create two task definitions (Fargate launch type):

**`myapp-backend-task`**
- Container: image = your `myapp-backend` ECR URI
- Port mapping: `4000`
- CPU/Memory: 0.25 vCPU / 0.5 GB is enough to start

**`myapp-frontend-task`**
- Container: image = your `myapp-frontend` ECR URI
- Port mapping: `80`
- Same CPU/Memory

---

## 5. Create the Application Load Balancer

1. EC2 → Load Balancers → **Create ALB** (internet-facing).
2. Create two target groups, type **IP** (required for Fargate):
   - `tg-backend` → port 4000 → health check path `/api/health`
   - `tg-frontend` → port 80 → health check path `/`
3. On the ALB listener (port 80, or 443 if you attach a cert via ACM):
   - **Default rule** → forward to `tg-frontend`
   - **Add rule**: if path is `/api/*` → forward to `tg-backend`

This is what makes "frontend + backend, one app" work behind a single
load balancer/domain — `/api/*` goes to Express, everything else goes
to the React app.

---

## 6. Create ECS Services

For each task definition, ECS → cluster → **Create Service**:
- Launch type: Fargate
- Desired tasks: 2 (for basic redundancy)
- Attach to the ALB:
  - backend service → `tg-backend`
  - frontend service → `tg-frontend`
- VPC/subnets: pick at least 2 public subnets (or private + NAT) and a
  security group that allows inbound on the container port from the ALB's
  security group only.

---

## 7. Point your domain at the ALB

Route 53 (or your DNS provider) → create an A/ALIAS record pointing your
domain to the ALB's DNS name.

---

## Notes
- Both containers run as **separate ECS services** so you can scale,
  redeploy, or roll back frontend and backend independently.
- For HTTPS, request a free cert in **ACM**, attach it to a port-443
  listener on the ALB, and redirect port 80 → 443.
- When you're ready for CI/CD instead of manual console pushes, this
  same structure maps cleanly onto GitHub Actions + `aws ecs update-service`,
  happy to set that up later.
