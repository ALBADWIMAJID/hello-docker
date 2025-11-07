# hello-docker
**Static page on NGINX in Docker → CI/CD via GitHub Actions → Kubernetes (Minikube).**  
_Русский: Статическая страница на NGINX, сборка/публикация образа и деплой в локальный кластер._

[![Build & Push](https://github.com/ALBADWIMAJID/hello-docker/actions/workflows/docker.yml/badge.svg)](https://github.com/ALBADWIMAJID/hello-docker/actions/workflows/docker.yml)
[![Deploy](https://github.com/ALBADWIMAJID/hello-docker/actions/workflows/deploy.yml/badge.svg)](https://github.com/ALBADWIMAJID/hello-docker/actions/workflows/deploy.yml)

**Live (static UI):** https://albadwimajid.github.io/hello-docker/  
**Docker Hub:** https://hub.docker.com/r/majid36344/hello-nginx

---

## 1) Overview
- **App:** single `index.html` served by **NGINX**.
- **Image:** `majid36344/hello-nginx` (tags used: `latest`, `1`, `2`, `3`).
- **CI/CD:** GitHub Actions builds & pushes the image; optional deploy job to Minikube via self-hosted runner.
- **Kubernetes:** `Deployment` + `Service (NodePort)`; rolling update & rollback verified.

---

## 2) Quick Start (Docker, local)
```bash
docker build -t majid36344/hello-nginx:dev .
docker run --rm -p 8080:80 majid36344/hello-nginx:dev
curl -I http://127.0.0.1:8080   # → HTTP/1.1 200 OK

3) Kubernetes (Minikube
# Start cluster (WSL)
minikube start --driver=docker --memory=2048mb

# Apply manifests
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml

# Get cluster IP and test NodePort
MINIKUBE_IP=$(minikube ip)
curl -I http://$MINIKUBE_IP:30080   # → HTTP/1.1 200 OK

# Observe rollout / scale
kubectl rollout status deployment/hello-nginx
kubectl scale deployment/hello-nginx --replicas=3
```
Manifests: k8s/deployment.yaml, k8s/service.yaml
(Contain CPU requests/limits + readiness/liveness probes.)

4) CI/CD (GitHub Actions)

Build & Push: .github/workflows/docker.yml — builds the image and pushes to Docker Hub using secrets:

DOCKERHUB_USERNAME

DOCKERHUB_TOKEN

Deploy (optional): .github/workflows/deploy.yml — runs on self-hosted runner and applies manifests using KUBE_CONFIG_DATA (or local kubeconfig on the runner).

Check the Actions tab for successful runs.


5) Rolling Update & Rollback (K8s)
```
   
# Update image (rolling update)
kubectl set image deployment/hello-nginx web=majid36344/hello-nginx:3
kubectl rollout status deployment/hello-nginx

# Roll back to previous
kubectl rollout undo deployment/hello-nginx
```
6) Autoscaling (HPA) — optional
```
minikube addons enable metrics-server
kubectl autoscale deployment hello-nginx --cpu=50% --min=1 --max=5
kubectl get hpa
```

7) Makefile (quality-of-life)
```
make build        # docker build -t majid36344/hello-nginx:latest .
make push         # docker push majid36344/hello-nginx:latest
make k8s-up       # apply k8s manifests
make status       # kubectl get deploy,svc,pods -o wide
make url          # prints MINIKUBE_IP + curl -I test
make k8s-down     # delete resources

```
8) Project Structure
```
.
├─ index.html
├─ Dockerfile
├─ k8s/
│  ├─ deployment.yaml
│  └─ service.yaml
├─ .github/workflows/
│  ├─ docker.yml
│  └─ deploy.yml        # optional
└─ Makefile             # optional
```
9) Proof of completion (what to show)

docker run hello-world and docker --version (screenshot).

Build + local run: curl -I http://localhost:8080 → 200 OK.

Docker Hub repo showing pushed tags.

GitHub Actions successful runs (Build/Deploy).

Minikube test: curl -I http://$(minikube ip):30080 → 200 OK.

Rolling update to :3 and rollback to :latest (logs/screens).

10) Security & Notes

Never commit secrets/keys. Use GitHub Secrets for CI.

.gitignore includes .ssh/, tokens, and env files.

For a temporary public URL (demo):
cloudflared tunnel --url "http://<MINIKUBE_IP>:30080"

Report: What we built & how (full summary)

Goal: build a Dockerized static site, push the image to Docker Hub, and deploy it on a local Kubernetes cluster (Minikube) with CI/CD and rollout/rollback.
Русский: Цель — собрать статический сайт в Docker, публиковать образ в Docker Hub и разворачивать в локальном Kubernetes (Minikube) с CI/CD и откатами.

1) Environment & source control

WSL / Ubuntu as the Linux environment.

Git + SSH to GitHub configured and verified:
```
ssh -T git@github.com  # "You've successfully authenticated"
```
Repository: ALBADWIMAJID/hello-docker (main).

Why: secure, repeatable environment; version control for all artifacts.

2) App & Docker image

Single, accessible index.html, served by NGINX.

Dockerfile copies the page into NGINX document root.

Local test:
```

docker build -t majid36344/hello-nginx:dev .
docker run --rm -p 8080:80 majid36344/hello-nginx:dev
curl -I http://127.0.0.1:8080  # → HTTP/1.1 200 OK

```
Image published to Docker Hub: docker.io/majid36344/hello-nginx (tags 1, 2, 3, latest).

Why: portable, reproducible artifact.

3) CI/CD (GitHub Actions)

Build & Push workflow (.github/workflows/docker.yml) builds image on push and publishes to Docker Hub using secrets.

(Optional) Deploy (.github/workflows/deploy.yml) runs on a self-hosted runner and applies K8s manifests via kubeconfig secret.

Outcome: automated build/publish; optional automated deploy. Proof in Actions.

4) Kubernetes (Minikube) deploy

Start cluster:
```
minikube start --driver=docker --memory=2048mb
```

Manifests in k8s/:

Deployment (imagePullPolicy: Always) + readiness/liveness probes (/) + CPU requests/limits (100m/250m).

Service type NodePort on 30080.
```
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl rollout status deployment/hello-nginx
curl -I http://$(minikube ip):30080  # → HTTP/1.1 200 OK
```

Why: real K8s deployment with health checks & stable access.

5) Scaling & rollout/rollback
```
kubectl scale deployment/hello-nginx --replicas=3
kubectl set image deployment/hello-nginx web=majid36344/hello-nginx:3
kubectl rollout status deployment/hello-nginx
kubectl rollout undo deployment/hello-nginx
```
Outcome: zero-downtime updates and safe rollback.

6) Autoscaling (HPA) — optional

```
minikube addons enable metrics-server
kubectl autoscale deployment hello-nginx --cpu=50% --min=1 --max=5
kubectl get hpa
```

Why: demonstrates resource-aware scaling.

7) Public links (demo)

Permanent static UI: https://albadwimajid.github.io/hello-docker/

Temporary live K8s URL (optional):
```

IP=$(minikube ip)
cloudflared tunnel --url "http://$IP:30080"
# → https://<random>.trycloudflare.com

```
8) Makefile (DX)
```
make build   # build image
make push    # push image
make k8s-up  # apply manifests
make status  # get deploy,svc,pods
make url     # NodePort URL test
make k8s-down

```
9) Grading checklist

✅ Docker works (hello-world, --version)

✅ Image builds & runs locally (HTTP 200)

✅ Pushed to Docker Hub (tags visible)

✅ GitHub Actions (Build/Deploy) green

✅ Minikube NodePort returns 200 OK

✅ Rolling update & rollback proven

✅ (Optional) HPA visible in kubectl get hpa

✅ Public links for demo (Pages / Tunnel)




