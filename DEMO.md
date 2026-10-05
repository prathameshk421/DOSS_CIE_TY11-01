# DEMO.md: DevOps CIE-01 Live Demo (Run on this Mac)

Flow being demonstrated:

```
Git (GitHub) -> Jenkins -> Docker image -> Kubernetes (Minikube) -> Prometheus -> Grafana
```

Everything runs locally on one Mac. Jenkins runs on the Mac itself (installed with Homebrew). Docker Desktop provides the Docker engine. Minikube creates a one-node Kubernetes cluster inside a Docker container.

> **Do not reinstall anything.** The tools (Docker, Minikube, kubectl, Jenkins) are already installed. This guide only resets what we *created* with them, then rebuilds it live.

## Ports used

| Port | What | URL |
|---|---|---|
| 8080 | Jenkins | http://localhost:8080 |
| random (Minikube prints it) | Website | printed by `minikube service` |
| 30090 | Prometheus (NodePort) | printed by `minikube service` |
| 3000 | Grafana (via port-forward) | http://localhost:3000 |

---

## Part 0: Before the demo: reset to scratch

Run these **before** the audience arrives. They delete what we created for this demo (our containers, our image, the cluster, the Jenkins job) but not the installed tools or other projects.

### 0.1 Delete the Kubernetes objects and the cluster

```bash
minikube delete
```

- `minikube delete` removes the Minikube cluster (the Docker container that acts as the Kubernetes node), together with every Deployment, Service, Pod and namespace inside it. This is the clean-slate command for Kubernetes.
- It keeps the cached Minikube files in `~/.minikube`, so the next `minikube start` is faster. Minikube stays installed.

### 0.2 Delete only this project's Docker containers and image

```bash
docker rm -f devops-demo devops-cie
```

- `docker rm` removes containers by name; `-f` (force) stops running ones first.
- Only the two containers we created in this demo are named. Containers from other projects are left alone.
- If a container does not exist, Docker prints an error for it. That is fine.

```bash
docker rmi -f devops-demo:latest
```

- `docker rmi` removes an image. `-f` forces removal even if a stopped container still references it.
- `devops-demo:latest` is the image name and tag we built from our `Dockerfile`.

Check it is gone:

```bash
docker ps -a | grep devops
docker images | grep devops-demo
```

- `grep` filters the lists. Both commands should print nothing.

> Do **not** run `docker system prune -af` or `docker rm -f $(docker ps -aq)` on this Mac. They delete every container and image, including other projects.

### 0.3 Delete the Jenkins job

1. Open http://localhost:8080
2. Click the job **devops-demo** and choose **Delete Pipeline** in the left sidebar, then confirm.

### 0.4 Make sure Jenkins is running (only the one service)

```bash
brew services list | grep jenkins
```

- `brew services list` shows background services managed by Homebrew.
- `grep jenkins` filters to the Jenkins line.
- It should say `jenkins  started`. If it does not:

```bash
brew services start jenkins
```

- `brew services start` launches the service now and registers it to start at login.
- Use `jenkins`, **not** `jenkins-lts`. Running both at once breaks Jenkins.

### 0.5 Make sure Docker Desktop is open

Start the Docker Desktop app and wait for the whale icon to stop animating. Confirm:

```bash
docker info > /dev/null && echo "Docker is up"
```

- `docker info` talks to the Docker daemon and fails if it is not running.
- `> /dev/null` throws away the long output.
- `&& echo ...` runs only if the previous command succeeded.

---

## Part 1: Git: where the code lives

**Say:** "Git stores our code and its history. GitHub is the remote copy that Jenkins watches."

```bash
cd ~/Code/DOSS_CIE_TY11-01
git status
git log --oneline -5
```

- `cd` moves into the project folder.
- `git status` shows whether files are modified, staged or clean.
- `git log` prints the commit history. `--oneline` shows one short line per commit; `-5` limits it to the last 5.

Show the files:

```bash
ls
```

| File | Purpose |
|---|---|
| `index.html` | The website |
| `Dockerfile` | Recipe for the Docker image |
| `Jenkinsfile` | The CI/CD pipeline as code |
| `deployment.yaml`, `service.yaml` | Kubernetes definitions of the app |
| `kubernetes_metrics.yaml`, `prometheus.yaml`, `grafana.yaml` | Monitoring stack |

---

## Part 2: Docker: package the website

**Say:** "Docker packages the website and a web server (nginx) into one portable image."

### 2.1 The Dockerfile

```dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
```

- `FROM nginx:alpine` is the base image: nginx on the small Alpine Linux.
- `COPY index.html ...` copies our page into nginx's web folder.
- `EXPOSE 80` documents that the container listens on port 80. It does not publish the port by itself.

### 2.2 Build the image

```bash
docker build -t devops-demo:latest .
```

- `docker build` builds an image from a Dockerfile.
- `-t devops-demo:latest` tags (names) the image `devops-demo` with the version tag `latest`.
- `.` is the build context: the current folder, where Docker finds the `Dockerfile` and `index.html`.

```bash
docker images
```

- Lists the local images. `devops-demo` should appear.

### 2.3 Run it as a container (quick check, outside Kubernetes)

```bash
docker run -d --name devops-demo -p 8081:80 devops-demo:latest
```

- `docker run` creates and starts a container from the image.
- `-d` (detached) runs it in the background.
- `--name devops-demo` gives the container a readable name.
- `-p 8081:80` publishes a port: host port **8081** forwards to container port **80**. We use 8081 because Jenkins already uses 8080.
- `devops-demo:latest` is the image to run.

```bash
docker ps
```

- Lists **running** containers (`-a` would show stopped ones too).

Open http://localhost:8081 to show the site, then clean up so the port is free:

```bash
docker rm -f devops-demo
```

- Stops (`-f`) and removes the container. The image stays.

---

## Part 3: Kubernetes (Minikube): the cluster

**Say:** "Kubernetes runs our container, keeps 2 copies alive, and exposes it. Minikube is a one-node Kubernetes cluster on this laptop."

### 3.1 Start the cluster

```bash
minikube start --driver=docker
```

- `minikube start` creates and boots the cluster.
- `--driver=docker` runs the cluster node as a Docker container (needs Docker Desktop running).
- Takes 1 to 2 minutes the first time after the reset.

```bash
kubectl get nodes
```

- `kubectl` is the command-line client for Kubernetes.
- `get nodes` lists the machines in the cluster. We expect one node, `minikube`, with status `Ready`.

### 3.2 Why we load the image manually

The cluster has its own Docker engine, separate from Docker Desktop's. Our image is not in a public registry, so we copy it in:

```bash
minikube image load devops-demo:latest
```

- `minikube image load` copies an image from your local Docker into the Minikube node so Kubernetes can use it.
- This is why `deployment.yaml` uses `imagePullPolicy: Never`: "never try to download it, use the one already on the node".

### 3.3 The manifests

`deployment.yaml` (key parts):

- `replicas: 2` asks Kubernetes to keep 2 identical pods running.
- `image: devops-demo:latest` is the image we loaded.
- `containerPort: 80` is the port nginx listens on.

`service.yaml` (key parts):

- `type: NodePort` exposes the app on a port of the node, so we can reach it from outside the cluster.
- `selector: app: devops-demo` routes traffic to pods carrying that label.

> You will **not** apply these by hand in the demo. Jenkins does it in Part 4. If the pipeline is not ready, the manual fallback is in "Troubleshooting" at the end.

---

## Part 4: Jenkins: automate it all

**Say:** "Jenkins watches Git. On every change it tests, builds, and deploys automatically. The pipeline is defined in the `Jenkinsfile`."

### 4.1 The Jenkinsfile, stage by stage

| Stage | Command | What it does |
|---|---|---|
| Checkout | `checkout scm` | Pulls the code from GitHub |
| Test HTML | `test -f index.html` | `test -f` succeeds only if the file exists |
| | `grep -qi "<html" index.html` | `grep` searches the file; `-q` quiet (just success or failure), `-i` ignore case. Fails if there is no `<html` tag |
| Build Docker Image | `docker build -t devops-demo:latest .` | Same as Part 2 |
| Load Image to Minikube | `minikube image load ...` | Same as Part 3.2 |
| Deploy to Kubernetes | `kubectl apply -f deployment.yaml` and `-f service.yaml` | `apply` creates or updates objects to match the file; `-f` is the file to read |
| | `kubectl rollout restart deployment/devops-demo-deployment` | Restarts the pods so they use the newly loaded `latest` image |
| Verify | `kubectl get pods` and `kubectl get service` | Shows that pods are running and the service exists |

Other lines in the file:

- `triggers { pollSCM('* * * * *') }` makes Jenkins check GitHub **every minute** and build if there is a new commit. (The five stars are cron: minute, hour, day, month, weekday.)
- `environment { PATH = "/opt/homebrew/bin:..." }` lets Jenkins find `docker`, `kubectl` and `minikube`, which Homebrew installs in `/opt/homebrew/bin`.

### 4.2 Create the job (live)

1. Open http://localhost:8080 and log in.
2. Click **New Item**, name it `devops-demo`, choose **Pipeline**, then **OK**.
3. Under **Pipeline** set:
   - **Definition:** Pipeline script from SCM
   - **SCM:** Git
   - **Repository URL:** `https://github.com/prathameshk421/DOSS_CIE_TY11-01`
   - **Branch Specifier:** `*/main`
   - **Script Path:** `Jenkinsfile`
4. Click **Save**, then **Build Now**.
5. Click build **#1**, then **Console Output**, and watch the stages go through. It should end with `Finished: SUCCESS`.

### 4.3 Verify the deployment

```bash
kubectl get pods
```

- Lists pods in the `default` namespace. We expect 2 pods named `devops-demo-deployment-...` with `STATUS Running` and `READY 1/1`.

```bash
kubectl get deployments
kubectl get service
```

- `deployments` shows desired and available replicas (`2/2`).
- `service` shows `devops-demo-service` of type `NodePort` and its port mapping.

### 4.4 Open the website

```bash
minikube service devops-demo-service --url
```

- `minikube service` finds the URL of a NodePort service.
- `--url` prints the URL instead of opening a browser.
- On macOS with the Docker driver this command **must stay running** in its terminal (it is a tunnel). Open the printed `http://127.0.0.1:<port>` in a browser.

---

## Part 5: The CI/CD moment: change, push, auto-deploy

**Say:** "Now I change the website, push to Git, and we do nothing else."

1. Edit `index.html` (for example change a heading).
2. Commit and push:

```bash
git add index.html
git commit -m "Demo: update page"
git push origin main
```

- `git add index.html` stages the file for the next commit.
- `git commit -m "..."` saves the staged changes; `-m` sets the message inline.
- `git push origin main` uploads the commit to the remote called `origin`, branch `main`.

3. Within about a minute Jenkins starts build **#2** by itself (polling). Watch it in the Jenkins UI.
4. Because the image tag stays `latest`, `kubectl apply` sees no change in the YAML and would leave the old pods running. The `Jenkinsfile` therefore runs this in the *Deploy to Kubernetes* stage:

```bash
kubectl rollout restart deployment/devops-demo-deployment
```

- `kubectl rollout restart` replaces the pods one by one (a rolling update), so they pick up the freshly loaded image.
- `deployment/devops-demo-deployment` is the object type and name.

Watch the rollout finish:

```bash
kubectl rollout status deployment/devops-demo-deployment
```

- Waits and prints progress until the rollout has finished.

5. Refresh the website. The change is visible.

---

## Part 6: Monitoring: Prometheus and Grafana

**Say:** "Prometheus collects metrics about the cluster, and Grafana draws them as graphs."

Three files, applied in this order:

| File | What it creates |
|---|---|
| `kubernetes_metrics.yaml` | `kube-state-metrics` (in `kube-system`): a small exporter that publishes Deployment metrics such as replica counts, plus the permissions (ServiceAccount, ClusterRole, ClusterRoleBinding) it needs to read them |
| `prometheus.yaml` | A `monitoring` namespace, Prometheus config, Prometheus Deployment, and a NodePort Service on port 30090. It scrapes `kube-state-metrics` every 15 seconds |
| `grafana.yaml` | Grafana Deployment and a ClusterIP Service on port 3000 |

### 6.1 Deploy them

```bash
kubectl apply -f kubernetes_metrics.yaml
kubectl apply -f prometheus.yaml
kubectl apply -f grafana.yaml
```

- `apply -f <file>` creates everything described in the file, or updates it if it already exists.

Wait until everything is running:

```bash
kubectl get pods -n monitoring
kubectl get pods -n kube-system -l app.kubernetes.io/name=kube-state-metrics
```

- `-n monitoring` (`--namespace`) limits the command to that namespace. Namespaces are folders that group objects.
- `-l key=value` (`--selector`) filters by label.
- Wait for `Running` and `1/1`. The first start downloads images, which can take a few minutes.

### 6.2 Open Prometheus

```bash
minikube service prometheus -n monitoring --url
```

- Same idea as Part 4.4. `-n monitoring` is needed because the Service lives in that namespace. Keep the terminal open.

In the browser:

1. Go to **Status → Targets** and check that `kube-state-metrics` is `UP`.
2. On the **Query** page run `kube_deployment_status_replicas_available` and press **Execute**.
   - It returns the number of available pods per Deployment, so `devops-demo-deployment` shows **2**.

### 6.3 Open Grafana

Grafana is `ClusterIP` (reachable only inside the cluster), so forward a local port to it:

```bash
kubectl port-forward -n monitoring service/grafana 3000:3000
```

- `kubectl port-forward` tunnels a local port to a service or pod inside the cluster.
- `-n monitoring` selects the namespace.
- `service/grafana` is the target.
- `3000:3000` is `<local port>:<service port>`.
- Keep this terminal open.

Open http://localhost:3000 and log in with the default `admin` / `admin`. Skip or change the password prompt.

### 6.4 Connect Grafana to Prometheus and make a graph

1. Left menu: **Connections → Data sources → Add data source → Prometheus**.
2. **URL:** `http://prometheus.monitoring.svc.cluster.local:9090`
   - This is the in-cluster DNS name: `<service>.<namespace>.svc.cluster.local:<port>`.
3. Click **Save & test**. It should say it succeeded.
4. **Dashboards → New → New dashboard → Add visualization**, choose the Prometheus data source, and enter the query `kube_deployment_status_replicas_available`.
5. Apply the panel. It shows 2 for `devops-demo-deployment`.

If the **Add visualization** button is not visible, click the blue **+** (Add new element) in the right-hand sidebar, or use the fallback: open **Explore**, choose the Prometheus data source, run the query, then click **Add to dashboard**.

### 6.5 (Nice finish) show the metric reacting live

```bash
kubectl scale deployment devops-demo-deployment --replicas=4
```

- `kubectl scale` changes the replica count.
- `--replicas=4` is the new desired number of pods.

Within about 15 to 30 seconds the Grafana panel climbs to 4. Then restore:

```bash
kubectl scale deployment devops-demo-deployment --replicas=2
```

---

## Part 7: Self-healing (optional crowd-pleaser)

```bash
kubectl get pods
kubectl delete pod <one-of-the-pod-names>
kubectl get pods
```

- `kubectl delete pod <name>` removes one pod.
- Kubernetes notices there are fewer than 2 and immediately starts a replacement. This is the point of a Deployment.

---

## Quick reference: flags used in this demo

| Flag | Meaning |
|---|---|
| `-t` (docker build) | Tag (name) the image |
| `-d` (docker run) | Detached: run in the background |
| `-p host:container` | Publish a port |
| `-a` (docker ps) | All (including stopped containers) |
| `-f` (docker rm / rmi) | Force |
| `-f` (kubectl apply) | **File** to read (different meaning!) |
| `-n` | Namespace |
| `-l` | Label selector |
| `--url` (minikube service) | Print the URL instead of opening a browser |
| `--driver=docker` | Run the Minikube node inside Docker |
| `--replicas=N` | Desired number of pods |

---

## Troubleshooting

| Problem | Fix |
|---|---|
| `Cannot connect to the Docker daemon` | Open Docker Desktop and wait for it to finish starting |
| `kubectl: connection refused` | Cluster is not running: `minikube start --driver=docker` |
| Pods in `ErrImagePull` / `ImagePullBackOff` | The image is not on the node: `minikube image load devops-demo:latest`, then `kubectl rollout restart deployment/devops-demo-deployment` |
| Jenkins `docker: command not found` | Restart Jenkins: `brew services restart jenkins` |
| Jenkins page blank or 500 | Only one Jenkins service must run. Check `brew services list`. Stop `jenkins-lts` if it is listed |
| Build #2 does not start | Open the job, then **Git Polling Log**. Check that you pushed to `main` |
| Browser says connection refused on the Minikube URL | The `minikube service ... --url` terminal was closed. Run it again |
| Prometheus target shows DOWN | Check `kubectl get pods -n kube-system` for the `kube-state-metrics` pod |
| Pipeline not ready, need a manual fallback | `docker build -t devops-demo:latest .` then `minikube image load devops-demo:latest` then `kubectl apply -f deployment.yaml -f service.yaml` |

## End of demo cleanup (optional)

```bash
kubectl delete -f grafana.yaml -f prometheus.yaml -f kubernetes_metrics.yaml -f service.yaml -f deployment.yaml
```

- `kubectl delete -f` removes every object defined in the listed files.

```bash
minikube stop
```

- Stops the cluster but keeps it, so it restarts quickly next time. Use `minikube delete` to remove it completely.
