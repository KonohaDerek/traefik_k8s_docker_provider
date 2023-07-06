
## 安裝 k3d

=Linux=
```bash
 > curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
```

=MAC=
```bash
 > brew install k3d
```

=Windows=
```bash
 > choco install k3d
```

## 建立 k3d cluster

```bash
 > k3d cluster create derek-cluster --port 80:32080@loadbalancer --port 443:32443@loadbalancer --port 8080:32090@loadbalancer  --k3s-arg "--disable=traefik@server:*" --k3s-arg "--disable=servicelb@server:*" --api-port 6550 --network reverse-proxy --volume /var/run/docker.sock:/var/run/docker.sock
```

## 取得 kubeconfig
```bash
> k3d kubeconfig get derek-cluster
```

## 安裝 Traefik

```
    # Install Traefik Resource Definitions:
 > kubectl apply -f https://raw.githubusercontent.com/traefik/traefik/v2.9/docs/content/reference/dynamic-configuration/kubernetes-crd-definition-v1.yml

    # Install RBAC for Traefik:
 > kubectl apply -f https://raw.githubusercontent.com/traefik/traefik/v2.9/docs/content/reference/dynamic-configuration/kubernetes-crd-rbac.yml
```
### 安裝 Traefik 並同時啟用 Docker Provider


``` bash
> kubectl apply -f traefik-deployments.yaml
```

```traefik-deployments.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  namespace: default
  name: traefik-ingress-controller

---
kind: Deployment
apiVersion: apps/v1
metadata:
  namespace: default
  name: traefik
  labels:
    app: traefik

spec:
  replicas: 1
  selector:
    matchLabels:
      app: traefik
  template:
    metadata:
      labels:
        app: traefik
    spec:
      serviceAccountName: traefik-ingress-controller
      containers:
        - name: traefik
          image: traefik:v2.9
          args:
            - --api.insecure
            - --accesslog
            - --entrypoints.web.Address=:8000
            - --entrypoints.websecure.Address=:4443
            - --api.dashboard=true
            - --providers.docker=true
            - --providers.docker.exposedbydefault=false
            - --providers.kubernetescrd
            - --certificatesresolvers.myresolver.acme.tlschallenge
            - --certificatesresolvers.myresolver.acme.email=your@mail.com
            - --certificatesresolvers.myresolver.acme.storage=acme.json
            # Please note that this is the staging Let's Encrypt server.
            # Once you get things working, you should remove that whole line altogether.
            - --certificatesresolvers.myresolver.acme.caserver=https://acme-staging-v02.api.letsencrypt.org/directory
            - --providers.file.filename=dynamic_conf.toml
            - --global.checknewversion
            - --global.sendanonymoususage
            - --ping=true
            - --metrics.prometheus=true
            - --metrics.prometheus.entrypoint=metrics
            - --entryPoints.metrics.address=:8082
            - --entryPoints.mongo.address=:27017
          ports:
            - name: web
              containerPort: 8000
            - name: websecure
              containerPort: 4443
            - name: admin
              containerPort: 8080
            - name: metrics
              containerPort: 8082
            - name: mongo
              containerPort: 27017
          volumeMounts:
            - name: dockersock
              mountPath: "/var/run/docker.sock"
          securityContext:
            privileged: true
            runAsUser: 0
      volumes:
        - name: dockersock
          hostPath:
            path: /var/run/docker.sock
          
```


``` bash
> kubectl apply -f traefik-service.yaml
```

```traefik-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: traefik

spec:
  type: NodePort
  ports:
    - protocol: TCP
      name: web
      port: 8000
      nodePort: 32080
    - protocol: TCP
      name: admin
      port: 8080
      nodePort: 32090
    - protocol: TCP
      name: websecure
      port: 4443
      nodePort: 32443
    - protocol: TCP
      name: metrics
      port: 8082
      nodePort: 32082
  selector:
    app: traefik
```


``` bash
> kubectl apply -f traefik-ingress.yaml
```

```traefik-ingress.yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: dashboard
spec:
  routes:
    - match: Host(`localhost`) && (PathPrefix(`/dashboard`) || PathPrefix(`/api`))
      kind: Rule
      services:
      - name: api@internal
        kind: TraefikService
          
```
