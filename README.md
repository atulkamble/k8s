# Kubernetes Core Objects: Pods, Services, Deployments, ConfigMaps & Secrets

Kubernetes uses objects to define, deploy, manage, configure, and expose containerized applications. These objects are typically created using YAML manifests and managed through `kubectl`.

## 1. Kubernetes objects overview

| Object     | Purpose                                               | Example                 |
| ---------- | ----------------------------------------------------- | ----------------------- |
| Pod        | Smallest deployable unit; runs one or more containers | Nginx container         |
| Deployment | Manages Pods, replicas, updates, and rollbacks        | 3 Nginx replicas        |
| Service    | Provides stable network access to Pods                | Expose Nginx on port 80 |
| ConfigMap  | Stores non-sensitive application configuration        | APP_ENV=production      |
| Secret     | Stores sensitive configuration data                   | Database password       |

## 2. Kubernetes architecture and object relationships

Conceptual view: Service routes traffic to Pods; Deployment manages replicas; Pods consume ConfigMaps and Secrets.

## 3. Understanding each object with YAML

### A. Pod

A Pod is the smallest deployable unit in Kubernetes. Containers inside a Pod share networking and can share storage volumes.

pod.yaml

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
    - name: nginx
      image: nginx:stable
      ports:
        - containerPort: 80
```

```
kubectl apply -f pod.yaml
kubectl get pods
kubectl describe pod nginx-pod
kubectl logs nginx-pod
kubectl delete pod nginx-pod
```

### B. Deployment

A Deployment manages application replicas through ReplicaSets and supports rolling updates, scaling, and rollbacks.

deployment.yaml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:stable
          ports:
            - containerPort: 80
```

```
kubectl apply -f deployment.yaml

kubectl get deployments
kubectl get replicasets
kubectl get pods

# Scale to 5 replicas
kubectl scale deployment nginx-deployment --replicas=5

# Update image
kubectl set image deployment/nginx-deployment \
  nginx=nginx:alpine

# Check rollout
kubectl rollout status deployment/nginx-deployment

# Rollback
kubectl rollout undo deployment/nginx-deployment
```

### C. Service

A Service exposes a group of Pods through a stable network endpoint, using labels and selectors.

| Service type | Purpose                                        |
| ------------ | ---------------------------------------------- |
| ClusterIP    | Internal cluster communication (default)       |
| NodePort     | Exposes a port on each node                    |
| LoadBalancer | Uses an external load balancer where supported |
| ExternalName | Maps a Service DNS name to another DNS name    |

service.yaml

```
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
```

```
kubectl apply -f service.yaml
kubectl get services
kubectl describe service nginx-service
kubectl get endpointslices \
  -l kubernetes.io/service-name=nginx-service
```

In AKS or EKS, a `LoadBalancer` Service can provision a cloud load balancer when the required integration is configured.

### D. ConfigMap

A ConfigMap stores non-confidential application configuration separately from container images.

configmap.yaml

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: "production"
  APP_NAME: "cloudnautic"
  LOG_LEVEL: "info"
```

```
kubectl apply -f configmap.yaml
kubectl get configmaps
kubectl describe configmap app-config
```

### E. Secret

A Secret stores sensitive values such as passwords, tokens, and certificates.

secret.yaml

```
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:
  DB_USER: "admin"
  DB_PASSWORD: "demo-password"
```

```
kubectl apply -f secret.yaml
kubectl get secrets
kubectl describe secret db-secret
```

`stringData` accepts plain text and Kubernetes converts it into Secret data. Secrets are not automatically encrypted securely at rest in every cluster. Use RBAC, encryption at rest, and an external secret manager for production workloads. Avoid committing real credentials to Git.

## 4. Using ConfigMaps and Secrets inside a Deployment

The following container configuration can be placed inside the Deployment's `containers` entry:

```
containers:
  - name: nginx
    image: nginx:stable
    envFrom:
      - configMapRef:
          name: app-config
      - secretRef:
          name: db-secret
    ports:
      - containerPort: 80
```

This injects the ConfigMap and Secret keys as environment variables when the container starts. Updating these values does not automatically refresh environment variables in running containers.

## 5. Comparison of Kubernetes objects

| Feature                   | Pod      | Deployment         | Service | ConfigMap | Secret |
| ------------------------- | -------- | ------------------ | ------- | --------- | ------ |
| Runs containers           | Yes      | Via Pods           | No      | No        | No     |
| Replica management        | No       | Yes                | No      | No        | No     |
| Networking endpoint       | Pod IP   | No stable endpoint | Yes     | No        | No     |
| Application configuration | Consumes | Defines references | No      | Yes       | Yes    |
| Sensitive data storage    | No       | No                 | No      | No        | Yes    |
| Rolling updates           | No       | Yes                | No      | No        | No     |

## 6. Hands-on lab: Deploy an Nginx application

Run the following commands in an existing Kubernetes cluster.

Step 1 — Create a Deployment

```
kubectl create deployment nginx --image=nginx:stable --replicas=3
```

Step 2 — Expose the Deployment

```
kubectl expose deployment nginx \
  --type=LoadBalancer \
  --port=80 \
  --target-port=80
```

Step 3 — Create a ConfigMap

```
kubectl create configmap app-config \
  --from-literal=APP_ENV=production
```

Step 4 — Create a Secret

```
kubectl create secret generic db-secret \
  --from-literal=DB_PASSWORD=demo-password
```

Step 5 — Verify resources

```
kubectl get pods,deployments,services
kubectl get configmaps,secrets
kubectl get all
```

Note: Steps 3 and 4 create configuration resources; they are not yet connected to the Deployment. Use the `envFrom` configuration shown above to connect them. Also, `kubectl get all` does not list every resource type, including ConfigMaps and Secrets.

## 7. Important concepts for interviews and practical training

- Pod vs Deployment: A Pod runs containers; a Deployment maintains the desired number of Pods.
- Deployment vs Service: Deployment manages application lifecycle; Service provides stable network access.
- ConfigMap vs Secret: ConfigMap holds non-sensitive settings; Secret holds sensitive settings.
- Self-healing: Deployment controllers replace failed or deleted Pods to maintain the desired replica count.
- Scaling: Increase or decrease Deployment replicas without manually creating individual Pods.
- Service discovery: Kubernetes DNS lets applications communicate using Service names.
- Labels and selectors: Services identify matching Pods, and Deployments identify their managed ReplicaSets and Pods.

Recommended learning sequence: Pod → ReplicaSet → Deployment → Service → ConfigMap → Secret → Namespace → Ingress → StatefulSet → DaemonSet → HPA.

Official reference: [Kubernetes Objects Documentation](https://kubernetes.io/docs/concepts/overview/working-with-objects/).
