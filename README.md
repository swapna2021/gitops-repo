# Welcome App with FluxCD GitOps

This project demonstrates deploying a simple React "Welcome App" using GitOps with FluxCD.

---

## 1. Prerequisites

Make sure you have the following installed on your machine:

- **Node.js** (>= 18)
- **Docker**
- **Git**
- **kubectl**
- **Flux CLI**
- **Kubernetes cluster** (Kind or Minikube recommended)

Install tools (macOS example):

```bash
brew install node kubectl fluxcd/tap/flux kind minikube
```

---

## 2. Create the React App

```bash
npx create-react-app welcome-app
cd welcome-app
```

Edit `src/App.js`:

```jsx
import React from "react";
import "./App.css";

export default function App() {
  return (
    <div className="App">
      <h1>👋 Welcome to FluxCD Demo</h1>
      <p>Deployed with GitOps using FluxCD.</p>
    </div>
  );
}
```

Run locally:

```bash
npm start
```

---

## 3. Dockerize the App

Create a multi-stage `Dockerfile`:

```dockerfile
# -------- Stage 1: Build React app --------
FROM node:18-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# -------- Stage 2: Serve with Nginx --------
FROM nginx:alpine
WORKDIR /usr/share/nginx/html
RUN rm -rf ./*
COPY --from=build /app/build .
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

Add `.dockerignore`:

```
node_modules
build
.git
.DS_Store
```

Build & push image:

```bash
docker build -t <dockerhub-username>/welcome-app:v1 .
docker push <dockerhub-username>/welcome-app:v1
```

---

## 4. GitOps Repository Structure

Create a separate repo called **gitops-repo** for Flux to manage:

```
gitops-repo/
├─ apps/
│  └─ welcome-app/
│     ├─ base/
│     │  ├─ deployment.yaml
│     │  ├─ service.yaml
│     │  └─ kustomization.yaml
│     └─ overlays/
│        └─ dev/
│           ├─ kustomization.yaml
│           └─ image-patch.yaml
└─ clusters/
   └─ dev-cluster/
      └─ kustomization.yaml
```

### Base Deployment (`apps/welcome-app/base/deployment.yaml`)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: welcome-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: welcome-app
  template:
    metadata:
      labels:
        app: welcome-app
    spec:
      containers:
        - name: web
          image: nginx
          ports:
            - containerPort: 80
```

### Base Service (`apps/welcome-app/base/service.yaml`)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: welcome-app
spec:
  selector:
    app: welcome-app
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
```

### Base Kustomization (`apps/welcome-app/base/kustomization.yaml`)
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
```

### Overlay Patch (`apps/welcome-app/overlays/dev/image-patch.yaml`)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: welcome-app
spec:
  template:
    spec:
      containers:
        - name: web
          image: <dockerhub-username>/welcome-app:v1
```

### Overlay Kustomization (`apps/welcome-app/overlays/dev/kustomization.yaml`)
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namePrefix: dev-
namespace: default
resources:
  - ../../base
patches:
  - path: image-patch.yaml
    target:
      kind: Deployment
      name: welcome-app
```

### Cluster Kustomization (`clusters/dev-cluster/kustomization.yaml`)
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../apps/welcome-app/overlays/dev
```

---

## 5. Push to GitHub

```bash
cd gitops-repo
git init
git add .
git commit -m "init: welcome-app GitOps"
git branch -M master
git remote add origin https://github.com/<your-username>/gitops-repo.git
git push -u origin master
```

---

## 6. Create a Kubernetes Cluster

Using Kind:

```bash
kind create cluster --name dev-cluster
kubectl get nodes
```

Using Minikube:

```bash
minikube start
kubectl get nodes
```

---

## 7. Bootstrap Flux

```bash
export GITHUB_TOKEN=<your-personal-access-token>

flux check --pre

flux bootstrap github   --owner=<your-username>   --repository=gitops-repo   --branch=master   --path=clusters/dev-cluster   --personal
```

---

## 8. Verify Deployment

Check resources:

```bash
kubectl get deploy,svc -A | grep welcome-app
```

Port-forward to access app:

```bash
kubectl port-forward svc/dev-welcome-app 8080:80
# Open http://localhost:8080
```

---

## 9. Update App

- Update code in CRA project.
- Build & push a new Docker image (`v2`, `v3`, …).
- Edit `apps/welcome-app/overlays/dev/image-patch.yaml` to use the new tag.
- Commit & push changes to `gitops-repo`.

Flux will reconcile and roll out automatically.

---

## 10. Next Steps

- Add Flux Image Automation for automatic image tag updates.
- Deploy to multiple clusters/environments with overlays (`staging`, `prod`).
- Set up monitoring (Prometheus + Grafana) and alerting.

---

✨ That’s it! You now have a fully GitOps-managed React app running on Kubernetes with FluxCD.
