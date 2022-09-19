# Prerequisite

1. Create a cluster with a registry

```
k3d cluster create demo --k3s-arg "--no-deploy=traefik@server:*" --registry-create demo-registry:0.0.0.0:12345 --port 7080:8080@loadbalancer
```

2. Create a namespace and make it default

```
kubectl create namespace example-api
kubectl ns example-api

```

3. Build and push a Docker image (it is assumed that current folder is `.k8s`)


```
docker build --file ../api/docker/Dockerfile.prod -t localhost:12345/example-api:0.1.0 ../api
docker push localhost:12345/example-api:0.1.0
```

# Full

```
cd ./full
kubectl apply -f db-config.yaml
kubectl apply -f db-secret.yaml
kubectl apply -f db-service.yaml
kubectl apply -f api-service.yaml
kubectl apply -f db-statefulset.yaml
kubectl apply -f api-deployment.yaml
cd ..
```