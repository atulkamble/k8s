# Kubernetes Deployments, ReplicaSets, StatefulSets, Ingress Controllers & CoreDNS

A practical guide covering architecture, YAML manifests, CLI commands, traffic flow, rolling updates, rollbacks, and hands-on labs.

## 1. Kubernetes architecture and traffic flow

Conceptual application architecture. The Ingress Controller routes HTTP traffic to Services, which forward it to matching Pods. CoreDNS supports internal DNS resolution.

## 2. Important Kubernetes objects

| Object             | Purpose                                 | Example                                 |
| ------------------ | --------------------------------------- | --------------------------------------- |
| Pod                | Runs containers                         | NGINX container                         |
| ReplicaSet         | Maintains desired Pod count             | 3 replicas                              |
| Deployment         | Manages stateless workloads and updates | Web application                         |
| StatefulSet        | Manages stateful workloads              | MySQL, MongoDB                          |
| Service            | Stable network access to Pods           | ClusterIP                               |
| Ingress            | HTTP/HTTPS routing rules                | `/app`, `/api`                          |
| Ingress Controller | Implements Ingress rules                | ingress-nginx                           |
| CoreDNS            | Resolves internal DNS names             | `web-service.default.svc.cluster.local` |

## 3. Deployments: Creating a scalable application

A Deployment manages ReplicaSets and provides declarative updates, self-healing, scaling, and rollbacks.

Create `deployment.yaml`:

```

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
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
          image: nginx:1.27
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 100m
              memory: 64Mi
            limits:
              cpu: 500m
              memory: 128Mi
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 2
            periodSeconds: 5

```

Run these commands:

```

kubectl apply -f deployment.yaml

kubectl get deployments
kubectl get replicasets
kubectl get pods -o wide

kubectl describe deployment nginx-deployment

kubectl rollout status deployment/nginx-deployment

```

## 4. ReplicaSets and self-healing

If one Pod is deleted, the ReplicaSet creates a replacement to maintain the desired count.

```

# Check replicas
kubectl get rs

# Delete one Pod (replace with actual Pod name)
kubectl delete pod POD_NAME

# Observe automatic replacement
kubectl get pods -w

# Increase replicas
kubectl scale deployment nginx-deployment --replicas=5

# Verify
kubectl get pods
kubectl get rs

```

Important: A ReplicaSet replaces failed or deleted Pods. A Deployment manages ReplicaSet creation and updates. Neither automatically guarantees application-level recovery unless health probes and suitable configurations are provided.

## 5. Rolling updates and rollbacks

A rolling update gradually replaces old Pods with new ones while respecting availability constraints.

\#chatgpt-mermaid-\_r_ut\_{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:16px;fill:rgb(13, 13, 13);}@keyframes edge-animation-frame{from{stroke-dashoffset:0;}}@keyframes dash{to{stroke-dashoffset:0;}}#chatgpt-mermaid-\_r_ut\_ .edge-animation-slow{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 50s linear infinite;stroke-linecap:round;}#chatgpt-mermaid-\_r_ut\_ .edge-animation-fast{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 20s linear infinite;stroke-linecap:round;}#chatgpt-mermaid-\_r_ut\_ .error-icon{fill:rgb(243, 243, 243);}#chatgpt-mermaid-\_r_ut\_ .error-text{fill:rgb(13, 13, 13);stroke:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_ut\_ .edge-thickness-normal{stroke-width:1px;}#chatgpt-mermaid-\_r_ut\_ .edge-thickness-thick{stroke-width:3.5px;}#chatgpt-mermaid-\_r_ut\_ .edge-pattern-solid{stroke-dasharray:0;}#chatgpt-mermaid-\_r_ut\_ .edge-thickness-invisible{stroke-width:0;fill:none;}#chatgpt-mermaid-\_r_ut\_ .edge-pattern-dashed{stroke-dasharray:3;}#chatgpt-mermaid-\_r_ut\_ .edge-pattern-dotted{stroke-dasharray:2;}#chatgpt-mermaid-\_r_ut\_ .marker{fill:rgb(143, 143, 143);stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_ut\_ .marker.cross{stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_ut\_ svg{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:16px;}#chatgpt-mermaid-\_r_ut\_ p{margin:0;}#chatgpt-mermaid-\_r_ut\_ .label{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_ut\_ .cluster-label text{fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_ut\_ .cluster-label span{color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_ut\_ .cluster-label span p{background-color:transparent;}#chatgpt-mermaid-\_r_ut\_ .label text,#chatgpt-mermaid-\_r_ut\_ span{fill:rgb(13, 13, 13);color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_ut\_ .node rect,#chatgpt-mermaid-\_r_ut\_ .node circle,#chatgpt-mermaid-\_r_ut\_ .node ellipse,#chatgpt-mermaid-\_r_ut\_ .node polygon,#chatgpt-mermaid-\_r_ut\_ .node path{fill:rgb(222, 234, 251);stroke:rgb(83, 154, 248);stroke-width:1px;}#chatgpt-mermaid-\_r_ut\_ .rough-node .label text,#chatgpt-mermaid-\_r_ut\_ .node .label text,#chatgpt-mermaid-\_r_ut\_ .image-shape .label,#chatgpt-mermaid-\_r_ut\_ .icon-shape .label{text-anchor:middle;}#chatgpt-mermaid-\_r_ut\_ .node .katex path{fill:#000;stroke:#000;stroke-width:1px;}#chatgpt-mermaid-\_r_ut\_ .rough-node .label,#chatgpt-mermaid-\_r_ut\_ .node .label,#chatgpt-mermaid-\_r_ut\_ .image-shape .label,#chatgpt-mermaid-\_r_ut\_ .icon-shape .label{text-align:center;}#chatgpt-mermaid-\_r_ut\_ .node.clickable{cursor:pointer;}#chatgpt-mermaid-\_r_ut\_ .root .anchor path{fill:rgb(143, 143, 143)!important;stroke-width:0;stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_ut\_ .arrowheadPath{fill:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_ut\_ .edgePath .path{stroke:rgb(143, 143, 143);stroke-width:1px;}#chatgpt-mermaid-\_r_ut\_ .flowchart-link{stroke:rgb(143, 143, 143);fill:none;}#chatgpt-mermaid-\_r_ut\_ .edgeLabel{background-color:rgb(252, 252, 252);text-align:center;}#chatgpt-mermaid-\_r_ut\_ .edgeLabel p{background-color:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_ut\_ .edgeLabel rect{opacity:0.5;background-color:rgb(252, 252, 252);fill:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_ut\_ .labelBkg{background-color:rgba(252, 252, 252, 0.5);}#chatgpt-mermaid-\_r_ut\_ .cluster rect{fill:rgb(243, 243, 243);stroke:rgba(0, 0, 0, 0.1);stroke-width:1px;}#chatgpt-mermaid-\_r_ut\_ .cluster text{fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_ut\_ .cluster span{color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_ut\_ div.mermaidTooltip{position:absolute;text-align:center;max-width:200px;padding:2px;font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:12px;background:rgb(243, 243, 243);border:1px solid rgba(0, 0, 0, 0.1);border-radius:2px;pointer-events:none;z-index:100;}#chatgpt-mermaid-\_r_ut\_ .flowchartTitleText{text-anchor:middle;font-size:18px;fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_ut\_ rect.text{fill:none;stroke-width:0;}#chatgpt-mermaid-\_r_ut\_ .icon-shape,#chatgpt-mermaid-\_r_ut\_ .image-shape{background-color:rgb(252, 252, 252);text-align:center;}#chatgpt-mermaid-\_r_ut\_ .icon-shape p,#chatgpt-mermaid-\_r_ut\_ .image-shape p{background-color:rgb(252, 252, 252);padding:2px;}#chatgpt-mermaid-\_r_ut\_ .icon-shape .label rect,#chatgpt-mermaid-\_r_ut\_ .image-shape .label rect{opacity:0.5;background-color:rgb(252, 252, 252);fill:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_ut\_ .label-icon{display:inline-block;height:1em;overflow:visible;vertical-align:-0.125em;}#chatgpt-mermaid-\_r_ut\_ .node .label-icon path{fill:currentColor;stroke:revert;stroke-width:revert;}#chatgpt-mermaid-\_r_ut\_ .node .neo-node{stroke:rgb(83, 154, 248);}#chatgpt-mermaid-\_r_ut\_ [data-look="neo"].node rect,#chatgpt-mermaid-\_r_ut\_ [data-look="neo"].cluster rect,#chatgpt-mermaid-\_r_ut\_ [data-look="neo"].node polygon{stroke:url(#chatgpt-mermaid-\_r_ut\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_ut\_ [data-look="neo"].swimlane.cluster rect{filter:none;}#chatgpt-mermaid-\_r_ut\_ [data-look="neo"].node path{stroke:url(#chatgpt-mermaid-\_r_ut\_-gradient);stroke-width:1px;}#chatgpt-mermaid-\_r_ut\_ [data-look="neo"].node .outer-path{filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_ut\_ [data-look="neo"].node .neo-line path{stroke:rgb(83, 154, 248);filter:none;}#chatgpt-mermaid-\_r_ut\_ [data-look="neo"].node circle{stroke:url(#chatgpt-mermaid-\_r_ut\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_ut\_ [data-look="neo"].node circle .state-start{fill:#000000;}#chatgpt-mermaid-\_r_ut\_ [data-look="neo"].icon-shape .icon{fill:url(#chatgpt-mermaid-\_r_ut\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_ut\_ [data-look="neo"].icon-shape .icon-neo path{stroke:url(#chatgpt-mermaid-\_r_ut\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_ut\_ .node text{font-size:14px;font-weight:600;letter-spacing:normal;fill:rgb(0, 79, 153);}#chatgpt-mermaid-\_r_ut\_ .edgeLabels text{font-size:13px;font-weight:600;letter-spacing:-0.08px;fill:rgb(0, 79, 153);}#chatgpt-mermaid-\_r_ut\_ .node tspan[font-weight="normal"],#chatgpt-mermaid-\_r_ut\_ .edgeLabels tspan[font-weight="normal"]{font-weight:600;}#chatgpt-mermaid-\_r_ut\_ .edgeLabel .label rect{opacity:1;rx:13px;ry:13px;fill:rgb(245, 250, 255);stroke:rgb(206, 219, 229);stroke-width:1px;}#chatgpt-mermaid-\_r_ut\_ .node rect,#chatgpt-mermaid-\_r_ut\_ .node circle,#chatgpt-mermaid-\_r_ut\_ .node ellipse,#chatgpt-mermaid-\_r_ut\_ .node polygon,#chatgpt-mermaid-\_r_ut\_ .node path{fill:rgb(229, 243, 255);stroke:rgba(0, 0, 0, 0.1);stroke-width:1px;}#chatgpt-mermaid-\_r_ut\_ .node rect{rx:16px;ry:16px;}#chatgpt-mermaid-\_r_ut\_ .node.mermaid-decision .label-container{fill:rgb(245, 250, 255);stroke:rgb(206, 219, 229);stroke-dasharray:2,2;}#chatgpt-mermaid-\_r_ut\_ .edgePaths .flowchart-link{stroke:rgb(143, 143, 143);stroke-width:1px;stroke-linecap:round;stroke-linejoin:round;}#chatgpt-mermaid-\_r_ut\_ .marker{fill:rgb(143, 143, 143);stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_ut\_ :root{--mermaid-font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";}Deployment v13 PodsRolling UpdateMixed v1 + v2PodsDeployment v23 PodsHealthy?Update CompleteManual RollbackYesNo

### Rolling update commands

```

# Update image
kubectl set image deployment/nginx-deployment \
  nginx=nginx:1.28

# Monitor rollout
kubectl rollout status deployment/nginx-deployment

# Check revision history
kubectl rollout history deployment/nginx-deployment

# View Deployment details
kubectl describe deployment nginx-deployment

# Roll back to previous revision
kubectl rollout undo deployment/nginx-deployment

# Roll back to specific revision
kubectl rollout undo deployment/nginx-deployment \
  --to-revision=1

# Pause and resume rollout
kubectl rollout pause deployment/nginx-deployment
kubectl rollout resume deployment/nginx-deployment

```

Kubernetes Deployments do not automatically roll back every failed update. The `kubectl rollout undo` command performs a rollback explicitly.

## 6. Deployments vs ReplicaSets vs StatefulSets

| Feature            | Deployment     | ReplicaSet           | StatefulSet                 |
| ------------------ | -------------- | -------------------- | --------------------------- |
| Main purpose       | Stateless apps | Pod replica count    | Stateful apps               |
| Pod identity       | Replaceable    | Replaceable          | Stable                      |
| Persistent storage | Supported      | Supported            | Per-Pod PVC templates       |
| Rolling updates    | Yes            | Not directly managed | Yes                         |
| Rollback history   | Yes            | No                   | Controller revision history |
| Scaling            | Yes            | Yes                  | Yes                         |
| Typical use        | Web API        | Internal controller  | Database                    |

### StatefulSet example

Create `statefulset.yaml`:

```

apiVersion: v1
kind: Service
metadata:
  name: nginx-headless
spec:
  clusterIP: None
  selector:
    app: nginx-stateful
  ports:
    - port: 80
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nginx-stateful
spec:
  serviceName: nginx-headless
  replicas: 3
  selector:
    matchLabels:
      app: nginx-stateful
  template:
    metadata:
      labels:
        app: nginx-stateful
    spec:
      containers:
        - name: nginx
          image: nginx:stable
          ports:
            - containerPort: 80

```

```

kubectl apply -f statefulset.yaml

kubectl get statefulsets
kubectl get pods -l app=nginx-stateful

kubectl scale statefulset nginx-stateful --replicas=5

```

The Pods have predictable identities such as `nginx-stateful-0`, `nginx-stateful-1`, and `nginx-stateful-2`. This example demonstrates stable identity; persistent storage requires a `volumeClaimTemplates` configuration.

## 7. Kubernetes Ingress and Ingress Controllers

Ingress defines routing rules for HTTP/HTTPS traffic. An Ingress Controller implements those rules and forwards requests to backend Services.

Host-based routing

### Prerequisite: Install an Ingress Controller

For a local Minikube cluster:

```

minikube addons enable ingress

kubectl get pods -n ingress-nginx
kubectl get ingressclass

```

For AKS, EKS, or a kubeadm cluster, install a compatible controller or use a managed ingress solution. Creating an Ingress resource alone does not expose an application.

### Create two backend Services

Create `services.yaml`:

```

apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  type: ClusterIP
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 80

```

These Services require existing Pods with matching `app: frontend` and `app: backend` labels.

### Host-based routing manifest

Create `ingress-host.yaml`:

```

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: backend-service
                port:
                  number: 80

```

### Path-based routing manifest

Create `ingress-path.yaml`:

```

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: example.com
      http:
        paths:
          - path: /app
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: backend-service
                port:
                  number: 80

```

```

kubectl apply -f services.yaml
kubectl apply -f ingress-host.yaml
kubectl apply -f ingress-path.yaml

kubectl get ingress
kubectl describe ingress host-ingress
kubectl describe ingress path-ingress

kubectl get svc
kubectl get endpointslices

```

Ingress forwards the original request path by default. For `/app` and `/api`, configure the applications to handle those prefixes or use controller-specific rewriting.

To test using an Ingress Controller IP:

```

curl -H "Host: app.example.com" http://INGRESS_IP/
curl -H "Host: api.example.com" http://INGRESS_IP/

curl -H "Host: example.com" http://INGRESS_IP/app
curl -H "Host: example.com" http://INGRESS_IP/api

```

Replace `INGRESS_IP` with the reachable address of your controller. For HTTPS, configure a TLS certificate and Ingress TLS settings.

## 8. CoreDNS and Kubernetes service discovery

CoreDNS provides DNS-based service discovery inside the Kubernetes cluster.

For example, a Service named `backend-service` in the `default` namespace can be resolved using:

`backend-service.default.svc.cluster.local`

\#chatgpt-mermaid-\_r_117\_{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:16px;fill:rgb(13, 13, 13);}@keyframes edge-animation-frame{from{stroke-dashoffset:0;}}@keyframes dash{to{stroke-dashoffset:0;}}#chatgpt-mermaid-\_r_117\_ .edge-animation-slow{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 50s linear infinite;stroke-linecap:round;}#chatgpt-mermaid-\_r_117\_ .edge-animation-fast{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 20s linear infinite;stroke-linecap:round;}#chatgpt-mermaid-\_r_117\_ .error-icon{fill:rgb(243, 243, 243);}#chatgpt-mermaid-\_r_117\_ .error-text{fill:rgb(13, 13, 13);stroke:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_117\_ .edge-thickness-normal{stroke-width:1px;}#chatgpt-mermaid-\_r_117\_ .edge-thickness-thick{stroke-width:3.5px;}#chatgpt-mermaid-\_r_117\_ .edge-pattern-solid{stroke-dasharray:0;}#chatgpt-mermaid-\_r_117\_ .edge-thickness-invisible{stroke-width:0;fill:none;}#chatgpt-mermaid-\_r_117\_ .edge-pattern-dashed{stroke-dasharray:3;}#chatgpt-mermaid-\_r_117\_ .edge-pattern-dotted{stroke-dasharray:2;}#chatgpt-mermaid-\_r_117\_ .marker{fill:rgb(143, 143, 143);stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_117\_ .marker.cross{stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_117\_ svg{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:16px;}#chatgpt-mermaid-\_r_117\_ p{margin:0;}#chatgpt-mermaid-\_r_117\_ .label{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_117\_ .cluster-label text{fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_117\_ .cluster-label span{color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_117\_ .cluster-label span p{background-color:transparent;}#chatgpt-mermaid-\_r_117\_ .label text,#chatgpt-mermaid-\_r_117\_ span{fill:rgb(13, 13, 13);color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_117\_ .node rect,#chatgpt-mermaid-\_r_117\_ .node circle,#chatgpt-mermaid-\_r_117\_ .node ellipse,#chatgpt-mermaid-\_r_117\_ .node polygon,#chatgpt-mermaid-\_r_117\_ .node path{fill:rgb(222, 234, 251);stroke:rgb(83, 154, 248);stroke-width:1px;}#chatgpt-mermaid-\_r_117\_ .rough-node .label text,#chatgpt-mermaid-\_r_117\_ .node .label text,#chatgpt-mermaid-\_r_117\_ .image-shape .label,#chatgpt-mermaid-\_r_117\_ .icon-shape .label{text-anchor:middle;}#chatgpt-mermaid-\_r_117\_ .node .katex path{fill:#000;stroke:#000;stroke-width:1px;}#chatgpt-mermaid-\_r_117\_ .rough-node .label,#chatgpt-mermaid-\_r_117\_ .node .label,#chatgpt-mermaid-\_r_117\_ .image-shape .label,#chatgpt-mermaid-\_r_117\_ .icon-shape .label{text-align:center;}#chatgpt-mermaid-\_r_117\_ .node.clickable{cursor:pointer;}#chatgpt-mermaid-\_r_117\_ .root .anchor path{fill:rgb(143, 143, 143)!important;stroke-width:0;stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_117\_ .arrowheadPath{fill:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_117\_ .edgePath .path{stroke:rgb(143, 143, 143);stroke-width:1px;}#chatgpt-mermaid-\_r_117\_ .flowchart-link{stroke:rgb(143, 143, 143);fill:none;}#chatgpt-mermaid-\_r_117\_ .edgeLabel{background-color:rgb(252, 252, 252);text-align:center;}#chatgpt-mermaid-\_r_117\_ .edgeLabel p{background-color:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_117\_ .edgeLabel rect{opacity:0.5;background-color:rgb(252, 252, 252);fill:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_117\_ .labelBkg{background-color:rgba(252, 252, 252, 0.5);}#chatgpt-mermaid-\_r_117\_ .cluster rect{fill:rgb(243, 243, 243);stroke:rgba(0, 0, 0, 0.1);stroke-width:1px;}#chatgpt-mermaid-\_r_117\_ .cluster text{fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_117\_ .cluster span{color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_117\_ div.mermaidTooltip{position:absolute;text-align:center;max-width:200px;padding:2px;font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:12px;background:rgb(243, 243, 243);border:1px solid rgba(0, 0, 0, 0.1);border-radius:2px;pointer-events:none;z-index:100;}#chatgpt-mermaid-\_r_117\_ .flowchartTitleText{text-anchor:middle;font-size:18px;fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_117\_ rect.text{fill:none;stroke-width:0;}#chatgpt-mermaid-\_r_117\_ .icon-shape,#chatgpt-mermaid-\_r_117\_ .image-shape{background-color:rgb(252, 252, 252);text-align:center;}#chatgpt-mermaid-\_r_117\_ .icon-shape p,#chatgpt-mermaid-\_r_117\_ .image-shape p{background-color:rgb(252, 252, 252);padding:2px;}#chatgpt-mermaid-\_r_117\_ .icon-shape .label rect,#chatgpt-mermaid-\_r_117\_ .image-shape .label rect{opacity:0.5;background-color:rgb(252, 252, 252);fill:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_117\_ .label-icon{display:inline-block;height:1em;overflow:visible;vertical-align:-0.125em;}#chatgpt-mermaid-\_r_117\_ .node .label-icon path{fill:currentColor;stroke:revert;stroke-width:revert;}#chatgpt-mermaid-\_r_117\_ .node .neo-node{stroke:rgb(83, 154, 248);}#chatgpt-mermaid-\_r_117\_ [data-look="neo"].node rect,#chatgpt-mermaid-\_r_117\_ [data-look="neo"].cluster rect,#chatgpt-mermaid-\_r_117\_ [data-look="neo"].node polygon{stroke:url(#chatgpt-mermaid-\_r_117\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_117\_ [data-look="neo"].swimlane.cluster rect{filter:none;}#chatgpt-mermaid-\_r_117\_ [data-look="neo"].node path{stroke:url(#chatgpt-mermaid-\_r_117\_-gradient);stroke-width:1px;}#chatgpt-mermaid-\_r_117\_ [data-look="neo"].node .outer-path{filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_117\_ [data-look="neo"].node .neo-line path{stroke:rgb(83, 154, 248);filter:none;}#chatgpt-mermaid-\_r_117\_ [data-look="neo"].node circle{stroke:url(#chatgpt-mermaid-\_r_117\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_117\_ [data-look="neo"].node circle .state-start{fill:#000000;}#chatgpt-mermaid-\_r_117\_ [data-look="neo"].icon-shape .icon{fill:url(#chatgpt-mermaid-\_r_117\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_117\_ [data-look="neo"].icon-shape .icon-neo path{stroke:url(#chatgpt-mermaid-\_r_117\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_117\_ .node text{font-size:14px;font-weight:600;letter-spacing:normal;fill:rgb(0, 79, 153);}#chatgpt-mermaid-\_r_117\_ .edgeLabels text{font-size:13px;font-weight:600;letter-spacing:-0.08px;fill:rgb(0, 79, 153);}#chatgpt-mermaid-\_r_117\_ .node tspan[font-weight="normal"],#chatgpt-mermaid-\_r_117\_ .edgeLabels tspan[font-weight="normal"]{font-weight:600;}#chatgpt-mermaid-\_r_117\_ .edgeLabel .label rect{opacity:1;rx:13px;ry:13px;fill:rgb(245, 250, 255);stroke:rgb(206, 219, 229);stroke-width:1px;}#chatgpt-mermaid-\_r_117\_ .node rect,#chatgpt-mermaid-\_r_117\_ .node circle,#chatgpt-mermaid-\_r_117\_ .node ellipse,#chatgpt-mermaid-\_r_117\_ .node polygon,#chatgpt-mermaid-\_r_117\_ .node path{fill:rgb(229, 243, 255);stroke:rgba(0, 0, 0, 0.1);stroke-width:1px;}#chatgpt-mermaid-\_r_117\_ .node rect{rx:16px;ry:16px;}#chatgpt-mermaid-\_r_117\_ .node.mermaid-decision .label-container{fill:rgb(245, 250, 255);stroke:rgb(206, 219, 229);stroke-dasharray:2,2;}#chatgpt-mermaid-\_r_117\_ .edgePaths .flowchart-link{stroke:rgb(143, 143, 143);stroke-width:1px;stroke-linecap:round;stroke-linejoin:round;}#chatgpt-mermaid-\_r_117\_ .marker{fill:rgb(143, 143, 143);stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_117\_ :root{--mermaid-font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";}Frontend PodCoreDNSbackend-serviceBackend Pod 1Backend Pod 2DNS QueryService IPHTTP Request

### CoreDNS commands

```

# Inspect CoreDNS Pods
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Inspect DNS Service
kubectl get svc kube-dns -n kube-system

# View CoreDNS configuration
kubectl get configmap coredns -n kube-system -o yaml

# DNS troubleshooting
kubectl run dns-test --image=busybox:1.36 \
  --restart=Never -- sleep 3600

kubectl exec dns-test -- nslookup kubernetes.default

kubectl exec dns-test -- \
  nslookup backend-service.default.svc.cluster.local

# Clean up
kubectl delete pod dns-test

```

The second Service lookup requires `backend-service` to exist in the `default` namespace.

## 9. End-to-end traffic flow

| Step | Component          | Function                                        |
| ---- | ------------------ | ----------------------------------------------- |
| 1    | Client             | Sends request to `app.example.com`              |
| 2    | External DNS       | Resolves public hostname to ingress entry point |
| 3    | Load Balancer      | Forwards traffic to Ingress Controller          |
| 4    | Ingress Controller | Evaluates host/path routing rules               |
| 5    | Service            | Provides stable access to matching backend Pods |
| 6    | Pod                | Processes the HTTP request                      |
| 7    | CoreDNS            | Resolves internal Service DNS names when needed |

CoreDNS is not necessarily consulted for every incoming request. It is primarily used by in-cluster clients and components that need DNS resolution.

## 10. Troubleshooting commands

| Problem                 | Useful command                                                      |
| ----------------------- | ------------------------------------------------------------------- |
| Deployment unavailable  | `kubectl describe deployment nginx-deployment`                      |
| Pod crashing            | `kubectl logs POD_NAME --previous`                                  |
| Image pull failure      | `kubectl describe pod POD_NAME`                                     |
| Service has no backends | `kubectl get endpointslices`                                        |
| Ingress not working     | `kubectl describe ingress host-ingress`                             |
| DNS resolution failure  | `kubectl exec dns-test -- nslookup kubernetes.default`              |
| Failed rolling update   | `kubectl rollout status deployment/nginx-deployment`                |
| Controller issue        | `kubectl logs -n ingress-nginx deployment/ingress-nginx-controller` |

## 11. Recommended hands-on practice

Kubernetes practical checklist

0/10 completed

Create an NGINX Deployment with 3 replicas

Delete one Pod and verify self-healing

Scale Deployment from 3 to 5 replicas

Perform rolling update from NGINX 1.27 to 1.28

Rollback Deployment to the previous revision

Create a StatefulSet with 3 Pods

Create frontend and backend Services

Configure host-based and path-based Ingress

Verify CoreDNS resolution using nslookup

Troubleshoot failed Pods, Services and Ingress

Reset checklist

Recommended learning sequence: Deployment → ReplicaSet → Scaling → Rolling Update → Rollback → StatefulSet → Service → Ingress Controller → Ingress Rules → CoreDNS → End-to-End Troubleshooting.

For production Kubernetes environments, also study readiness and liveness probes, PodDisruptionBudgets, Horizontal Pod Autoscaling, TLS certificates, NetworkPolicies, and Gateway API. Gateway API is particularly important for newer Kubernetes traffic-routing designs.
