# My App — React + Express on AWS EKS with Load Balancer

A full-stack application with a React (Vite) frontend and Node/Express backend, containerized with Docker, pushed to Docker Hub, and deployed on AWS EKS (Kubernetes) behind an AWS Load Balancer.

---

## Live Screenshots

### App Running via AWS Load Balancer
![App Running](https://raw.githubusercontent.com/AkhiJAIN/k8s_project_frontend_backend_1/main/Screenshot%202026-07-01%20071005.png)

### Backend API Response
![Backend Response](https://raw.githubusercontent.com/AkhiJAIN/k8s_project_frontend_backend_1/main/Screenshot%202026-07-01%20071130.png)

---

## Project Structure

```
myapp/
  frontend/          React (Vite) app, served by Nginx in production
  backend/           Express API
  docker-compose.yml For local testing of both containers together
  k8s-manifests.yml  Kubernetes Deployments + Services
```

---

## Architecture

```
Internet
   │
   ▼
AWS Load Balancer (ELB)
   │
   ▼
Frontend Pods - Nginx (port 80)
   │  /api/*
   ▼
Backend Pods - Express (port 4000)
```

---

## 1. Run Locally

```bash
docker compose up --build
```

- Frontend: http://localhost:8080
- Backend:  http://localhost:4000/api/hello

---

## 2. Push Images to Docker Hub

```bash
# Login
docker login

# Build and push backend
cd backend
docker build -t <your-dockerhub-username>/myapp-backend:latest .
docker push <your-dockerhub-username>/myapp-backend:latest

# Build and push frontend
cd ../frontend
docker build -t <your-dockerhub-username>/myapp-frontend:latest .
docker push <your-dockerhub-username>/myapp-frontend:latest
```

---

## 3. Create EKS Cluster

```bash
eksctl create cluster \
  --name my-cluster \
  --region ap-northeast-1 \
  --nodegroup-name my-node \
  --node-type c7i-flex.large \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3 \
  --managed
```

---

## 4. Connect kubectl to the Cluster

```bash
aws eks update-kubeconfig --name my-cluster --region ap-northeast-1
```

---

## 5. Deploy to Kubernetes

```bash
kubectl apply -f k8s-manifests.yml
```

This creates:
- `backend` Deployment (2 replicas) + ClusterIP Service
- `frontend` Deployment (2 replicas) + LoadBalancer Service

---

## 6. Get the Live URL

```bash
kubectl get svc frontend-svc
```

Copy the `EXTERNAL-IP` — that is your live app URL served by the AWS Load Balancer.

---

## 7. Verify Everything is Running

```bash
kubectl get pods       # all 4 pods should show Running
kubectl get svc        # frontend-svc shows EXTERNAL-IP
```

---

## Tech Stack

| Layer       | Technology                  |
|-------------|----------------------------|
| Frontend    | React + Vite               |
| Backend     | Node.js + Express          |
| Web Server  | Nginx                      |
| Container   | Docker + Docker Hub        |
| Orchestration | Kubernetes (AWS EKS)     |
| Load Balancer | AWS ELB (auto-provisioned)|
| Cloud       | AWS (ap-northeast-1)       |

---

## Notes

- Frontend Nginx proxies `/api/*` to the backend service (`backend-svc:4000`) inside the cluster
- Backend stays internal (`ClusterIP`) — only the frontend is publicly exposed
- To update the app: rebuild the Docker image, push to Docker Hub, then run `kubectl rollout restart deployment/<name>`