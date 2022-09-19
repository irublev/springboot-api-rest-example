# Prerequisites

1. Create a cluster with a registry

```
k3d cluster create demo --k3s-arg "--no-deploy=traefik@server:*" --registry-create demo-registry:0.0.0.0:12345 --port 7080:8080@loadbalancer
```

2. Build and push a Docker image (it is assumed that current folder is `.k8s`)


```
docker build --file ../api/docker/Dockerfile.prod -t localhost:12345/example-api:0.1.0 ../api
docker push localhost:12345/example-api:0.1.0
```

3. Create a namespace and make it default

```
kubectl create namespace example-api
kubectl ns example-api

```

Each time it is necessary to remove previous resources to be able to deploy new ones, please execute

```
kubectl delete namespace example-api
kubectl create namespace example-api
```

# Deployment variants

## 1. Single pod with sidecar, no persistence due to ephemeral volume

```
cd ./1-single-pod-with-ephemeral-volume
kubectl apply -f api-pod.yaml
kubectl wait pod api-pod --for condition=Ready --timeout=90s
kubectl port-forward api-pod 7880:80
cd ..
```

### Disadvantages

- No persisence, data may be lost!
- Not scalable, containers may crash with stopping the whole pod
- Not secure, volume is shared between all containers
- Not possible to update and restart the app without relaunching database
- Very unstable way to expose the app and the latter is only to the host
- Configuration parameters are copy-pasted
- Password is hard-coded

## 2. Only pods, no persistence due to ephemeral volume

```
cd ./2-pods-with-ephemeral-volume
kubectl apply -f db-pod.yaml
kubectl wait pod database-pod --for condition=Ready --timeout=90s
DATABASE_POD_IP=$(kubectl get pod database-pod --template '{{.status.podIP}}')
cat api-pod.yaml | sed "s/<database-pod-ip>/${DATABASE_POD_IP//./-}/" | kubectl apply -f -
kubectl wait pod api-pod --for condition=Ready --timeout=90s
kubectl port-forward api-pod 7880:80
cd ..
```

### Advantages

- Now volume with database data belongs only to database
- Now crashing of one component does not lead immediately to stopping the other

### Disadvantages

- Still no persistence, data may be lost!
- Still not scalable
- If database is restarted, its pod IP changes, so we have to restart also the app,
much more complex way to connect the app to DB
- Still bad expose of the app
- Still much copy-pasting of parameters
- Password is hard-coded

## 5. Full

```
cd ./5-full
kubectl apply -f db-config.yaml
kubectl apply -f db-secret.yaml
kubectl apply -f db-service.yaml
kubectl apply -f api-service.yaml
kubectl apply -f db-statefulset.yaml
kubectl apply -f api-deployment.yaml
cd ..
```

### Advantages

- Both components are scalable
- Database data are persisted
- Very stable way of how the app communcates with DB
- Very stable and reliable way to expose the app outside the cluster, now it is possible to reach the app not only from the host
- Configuration parameters are not copy-pasted
- Password is in a secret, we can conceal it from some users (will be discussed later)
