# k8s-masterclass-labs

This repository is a ready-to-run lab kit for the 28-day Kubernetes masterclass (3 hours/day). It contains manifests, Makefile targets, and scripts so you can run each week's labs quickly using `kind` (default), `k3d` or `minikube`.

---

## Repo tree (top-level)

```
k8s-masterclass-labs/
├── Makefile
├── README.md
├── LICENSE
├── .github/
│   └── workflows/
│       └── ci-build.yml
├── bin/
│   ├── kind-create.sh
│   ├── kind-delete.sh
│   ├── deploy-week.sh
│   └── build-and-push.sh
├── labs/
│   ├── week01-basics/
│   │   ├── README.md
│   │   ├── kind-config.yaml
│   │   └── manifests/
│   │       ├── namespace.yaml
│   │       ├── backend-deploy.yaml
│   │       ├── backend-svc.yaml
│   │       ├── frontend-configmap.yaml
│   │       ├── frontend-deploy.yaml
│   │       └── frontend-svc.yaml
│   ├── week02-networking-security/
│   │   ├── README.md
│   │   └── manifests/
│   │       ├── calico-install.yaml
│   │       └── networkpolicy-isolate.yaml
│   ├── week03-observability-cicd/
│   │   ├── README.md
│   │   └── manifests/
│   │       ├── prometheus/
│   │       ├── grafana/
│   │       └── argo-apps/
│   ├── week04-production-readiness/
│   │   ├── README.md
│   │   └── manifests/
│   │       ├── hpa.yaml
│   │       ├── velero-config.yaml
│   │       └── etcd-backup-notes.md
│   └── final-project/
│       ├── README.md
│       └── helm-chart/ (skeleton)
└── examples/
    └── app/ (simple sample app used in CI)
        ├── Dockerfile
        └── server.py
```

---

## Quick start (local)

1. Clone:
```bash
git clone https://github.com/youruser/k8s-masterclass-labs.git
cd k8s-masterclass-labs
chmod +x bin/*.sh
```

2. Create a kind cluster (default):
```bash
make kind-create
# or
bin/kind-create.sh
```

3. Deploy Week 1 lab manifests:
```bash
make deploy-week WEEK=01
# or
bin/deploy-week.sh 01
```

4. Port-forward frontend:
```bash
kubectl -n demo port-forward deploy/frontend 8080:80
# open http://localhost:8080
```

5. When finished:
```bash
make kind-delete
# or
bin/kind-delete.sh
```

---

## Makefile (top-level)

```makefile
# Makefile - top-level convenience targets
CLUSTER_NAME ?= k8s-lab
KIND_CONFIG ?= labs/week01-basics/kind-config.yaml
WEEK ?= 01
IMAGE_REGISTRY ?= ghcr.io/youruser
IMAGE_NAME ?= sample-app

.PHONY: kind-create kind-delete deploy-week build-and-push

kind-create:
	./bin/kind-create.sh $(CLUSTER_NAME) $(KIND_CONFIG)

kind-delete:
	./bin/kind-delete.sh $(CLUSTER_NAME)

deploy-week:
	./bin/deploy-week.sh $(WEEK)

build-and-push:
	./bin/build-and-push.sh $(IMAGE_REGISTRY) $(IMAGE_NAME)

```

---

## bin/kind-create.sh

```bash
#!/usr/bin/env bash
set -euo pipefail
CLUSTER_NAME=${1:-k8s-lab}
KIND_CONFIG=${2:-labs/week01-basics/kind-config.yaml}

if ! command -v kind >/dev/null 2>&1; then
  echo "kind not found. Install kind: https://kind.sigs.k8s.io/"
  exit 1
fi

echo "Creating kind cluster '${CLUSTER_NAME}'..."
kind create cluster --name "${CLUSTER_NAME}" --config "${KIND_CONFIG}"

kubectl cluster-info --context kind-${CLUSTER_NAME}
```

---

## bin/kind-delete.sh

```bash
#!/usr/bin/env bash
set -euo pipefail
CLUSTER_NAME=${1:-k8s-lab}

if ! command -v kind >/dev/null 2>&1; then
  echo "kind not found."
  exit 1
fi

echo "Deleting kind cluster '${CLUSTER_NAME}'..."
kind delete cluster --name "${CLUSTER_NAME}"
```

---

## bin/deploy-week.sh

```bash
#!/usr/bin/env bash
set -euo pipefail
WEEK=${1:-01}
ROOT_DIR=$(cd "$(dirname "$0")/.." && pwd -P)

case "$WEEK" in
  01)
    kubectl apply -f ${ROOT_DIR}/labs/week01-basics/manifests
    ;;
  02)
    kubectl apply -f ${ROOT_DIR}/labs/week02-networking-security/manifests
    ;;
  03)
    kubectl apply -f ${ROOT_DIR}/labs/week03-observability-cicd/manifests
    ;;
  04)
    kubectl apply -f ${ROOT_DIR}/labs/week04-production-readiness/manifests
    ;;
  final)
    kubectl apply -f ${ROOT_DIR}/labs/final-project/helm-chart
    ;;
  *)
    echo "Unknown week '${WEEK}'."
    exit 2
    ;;
esac
```

---

## bin/build-and-push.sh (example: builds example/app and pushes to GHCR)

```bash
#!/usr/bin/env bash
set -euo pipefail
REGISTRY=${1:?}
NAME=${2:?}
TAG=${3:-latest}
ROOT_DIR=$(cd "$(dirname "$0")/.." && pwd -P)
cd ${ROOT_DIR}/examples/app

docker build -t ${REGISTRY}/${NAME}:${TAG} .
# push (login required)
docker push ${REGISTRY}/${NAME}:${TAG}
```

---

## labs/week01-basics/kind-config.yaml

```yaml
# minimal two-node cluster (control-plane + worker)
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
```

---

## labs/week01-basics/manifests/namespace.yaml

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo
```

## labs/week01-basics/manifests/backend-deploy.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: hashicorp/http-echo:0.2.3
        args:
        - "-text=hello from backend"
        ports:
        - containerPort: 5678
        readinessProbe:
          httpGet:
            path: /
            port: 5678
          initialDelaySeconds: 2
          periodSeconds: 3
```

## labs/week01-basics/manifests/backend-svc.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: demo
spec:
  selector:
    app: backend
  ports:
  - protocol: TCP
    port: 80
    targetPort: 5678
  type: ClusterIP
```

## labs/week01-basics/manifests/frontend-configmap.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-html
  namespace: demo
data:
  index.html: |
    <html><body>
    <h1>Frontend</h1>
    <p>Backend says: <span id="msg">not fetched</span></p>
    </body></html>
```

## labs/week01-basics/manifests/frontend-deploy.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: nginx:stable-alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: default-conf
          mountPath: /usr/share/nginx/html/index.html
          subPath: index.html
      volumes:
      - name: default-conf
        configMap:
          name: frontend-html
```

## labs/week01-basics/manifests/frontend-svc.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: demo
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
  type: NodePort
```

---

## labs/week02-networking-security/README.md (short)

Contains: Calico manifest for local install, sample NetworkPolicy to isolate `demo` namespace, RBAC sample to create a read-only role.

---

## labs/week02-networking-security/manifests/networkpolicy-isolate.yaml

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-by-default
  namespace: demo
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

(Use this as a starting lab: then add allow rules for frontend->backend only.)

---

## labs/week03-observability-cicd/README.md (short)

Contains skeleton manifests to install Prometheus (kube-prometheus-stack), Grafana, and ArgoCD application manifests to demonstrate GitOps sync. These are **large** charts — the README explains `helm repo add` and `helm install --namespace monitoring` steps.

---

## labs/week04-production-readiness/README.md (short)

Contains HPA example, Velero notes for backups, and a simulated chaos script (e.g., delete pods, cordon nodes) to practice incident response.

---

## labs/final-project/README.md

Step-by-step guide to build the final project. Includes a Helm chart skeleton and a checklist for production hardening (network policies, RBAC, TLS, monitoring, backups, CI/CD).

---

## examples/app/Dockerfile

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY server.py /app/server.py
CMD ["python", "server.py"]
```

## examples/app/server.py

```python
from http.server import BaseHTTPRequestHandler, HTTPServer

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.send_header('Content-type','text/plain')
        self.end_headers()
        self.wfile.write(b'hello from sample app')

if __name__ == '__main__':
    server = HTTPServer(('0.0.0.0', 8080), Handler)
    print('Listening on 8080')
    server.serve_forever()
```

---

## .github/workflows/ci-build.yml (example)

A simple workflow that builds and publishes the example Docker image to GHCR when a push to `main` occurs.

```yaml
name: CI Build
on:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up QEMU
        uses: docker/setup-qemu-action@v2
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2
      - name: Login to GHCR
        uses: docker/login-action@v2
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: ./examples/app
          push: true
          tags: ghcr.io/${{ github.repository_owner }}/sample-app:latest
```

---

## How to extend

- Add more labs under `labs/weekXX-*` following the layout used for week01.
- Put long Helm charts under `charts/` or use `helmfile` if preferred.
- Add `argocd` manifests to `labs/week03-observability-cicd/manifests/argo-apps` to practice GitOps.

---

## License
MIT

---

Happy labs! Use `make deploy-week WEEK=01` to start Week 1. The repository skeleton is intended to be copied into a real GitHub repo — replace `youruser` placeholders with your GitHub handle and update the `IMAGE_REGISTRY`.
