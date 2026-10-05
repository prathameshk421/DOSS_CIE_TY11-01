# Student 3 - Docker

## Role

Docker packages the website and nginx into a portable container image.

## Files Associated

| File | What it does |
|---|---|
| Dockerfile | Instructions to build the image |
| index.html | The file copied into the image |

## Dockerfile Explained

```dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
```

- `FROM` selects the base image.
- `COPY` copies index.html into the nginx web root.
- `EXPOSE` documents the port the container listens on.
- `RUN` is not used because nothing needs to be installed.
- `CMD` is not set because the nginx image already starts nginx.

## Setup

Install and start Docker Desktop, then check:

```bash
docker --version
docker ps
```

## Commands to Demonstrate

```bash
docker build -t devops-demo:latest .
docker images
docker run -d --name devops-demo -p 8081:80 devops-demo:latest
docker ps
docker logs devops-demo
```

Open `http://localhost:8081`.

Cleanup:

```bash
docker stop devops-demo
docker rm devops-demo
```

Port 8081 is used because Jenkins runs on 8080.

## Demo Flow

1. Show the Dockerfile.
2. Build the image and show it in `docker images`.
3. Run the container and show it in `docker ps`.
4. Open the website in the browser.
5. Show the container logs.

## Troubleshooting

Container runs but the website is not reachable:

```bash
docker ps
docker logs devops-demo
```

Check the container port, the host port mapping (`-p host:container`) and that nginx is listening on port 80.

## Viva Points

- An image is a read-only template; a container is a running instance of it.
- Alpine images are small.
- `-p 8081:80` maps host port 8081 to container port 80.
- `-d` runs the container in the background.
