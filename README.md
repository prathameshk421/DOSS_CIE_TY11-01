# DevOps CIE-01 Demo

A small static HTML website taken through the full DevOps flow:

Git → Jenkins → Docker → Kubernetes (Minikube) → Prometheus → Grafana

## Files

| File | Purpose |
|---|---|
| index.html | The website |
| Dockerfile | Builds an nginx image containing index.html |
| Jenkinsfile | Pipeline: checkout, test, build, load image, deploy, verify |
| deployment.yaml | Kubernetes Deployment with 2 replicas |
| service.yaml | Kubernetes NodePort Service |
| prometheus.yml | Prometheus scrape configuration |

## Flow

1. Code is pushed to Git.
2. Jenkins checks out the code and tests the HTML.
3. Jenkins builds the Docker image.
4. The image is loaded into Minikube.
5. Jenkins applies the Deployment and Service.
6. Kubernetes runs and exposes the website.
7. Prometheus collects metrics and Grafana visualizes them.

## Group Members

| Member | Tool | Guide |
|---|---|---|
| Student 1 | Git | [docs/student1-git.md](docs/student1-git.md) |
| Student 2 | Jenkins | [docs/student2-jenkins.md](docs/student2-jenkins.md) |
| Student 3 | Docker | [docs/student3-docker.md](docs/student3-docker.md) |
| Student 4 | Kubernetes | [docs/student4-kubernetes.md](docs/student4-kubernetes.md) |

## Quick Start

```bash
docker build -t devops-demo:latest .
minikube start --driver=docker
minikube image load devops-demo:latest
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
minikube service devops-demo-service
```
