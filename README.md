# ElectronicsWeb – End-to-End DevOps Implementation

## 1. Project Overview

This project demonstrates the implementation of an end-to-end DevOps workflow for a full-stack web application using Docker, GitHub Actions, GitHub Container Registry (GHCR), Harbor, Kubernetes and Minikube.

The objective is to containerize the ElectronicsWeb application, automate Docker image builds, publish container images to a registry, and deploy the application on a Kubernetes cluster.

The application consists of:

- **Frontend:** React.js with Vite
- **Backend:** Node.js and Express.js
- **Database:** MongoDB Atlas
- **Authentication:** JWT
- **Containerization:** Docker
- **CI:** GitHub Actions
- **Container Registry:** GitHub Container Registry (GHCR) and Harbor
- **Container Orchestration:** Kubernetes
- **Local Kubernetes Environment:** Minikube
- **Package Management:** Helm (planned)
- **GitOps:** Argo CD (planned)

### Project Objectives

1. Containerize the frontend and backend applications.
2. Automate Docker image creation using GitHub Actions.
3. Publish container images to GitHub Container Registry.
4. Set up a private Harbor container registry.
5. Transfer application images from GHCR to Harbor.
6. Configure Kubernetes to authenticate with a private registry.
7. Deploy the backend and frontend applications on Kubernetes.
8. Configure Kubernetes Services for internal and external communication.
9. Connect the frontend to the backend using Nginx reverse proxy.
10. Prepare the infrastructure for Helm-based deployments and Argo CD GitOps integration.

---

## 2. Architecture

### High-Level Architecture

The implementation uses two GitHub repositories:

- **ElectronicsWeb:** Contains the application source code.
- **Devops_Proj1:** Contains Docker configurations, GitHub Actions workflows and Kubernetes manifests.

The CI workflow checks out both repositories, prepares the Docker build contexts, builds the frontend and backend images, and pushes them to GHCR.

The images are subsequently transferred to the local Harbor registry and deployed on Minikube.

### Architecture Diagram

```text
                  Developer
                      |
                      |
                Git Push
                      |
                      v
             GitHub Repositories
             ___________________
            |                   |
            | ElectronicsWeb    |
            | Application Code  |
            |___________________|
                      |
                      |
            _____________________
           |                     |
           | Devops_Proj1        |
           | Docker / CI / K8s   |
           |_____________________|
                      |
                      v
              GitHub Actions
                      |
             __________________
            |                  |
            | Docker Build     |
            | Frontend Image   |
            | Backend Image    |
            |__________________|
                      |
                      v
             GitHub Container
                Registry
                   GHCR
                      |
                      | Manual Image Transfer
                      v
               Harbor Registry
              (Local Private Lab)
                      |
                      v
                Minikube Cluster
                      |
             _____________________
            |                     |
            |  Kubernetes         |
            |  electronicsweb     |
            |  Namespace          |
            |_____________________|
                      |
           ___________|____________
          |                        |
          v                        v
    Frontend Pod              Backend Pod
    React + Nginx             Node.js + Express
    Port 80                   Port 5000
          |                        |
          |                        |
          |    Backend Service     |
          |<-----------------------|
          |                        |
          |                   MongoDB Atlas
          |                   External Database
          |
       Browser Access
       NodePort 30080
```

### Application Request Flow

1. A user accesses the application through the frontend Kubernetes NodePort Service.
2. The request reaches the React application served by Nginx.
3. When the frontend makes an API request, it uses the `/api` path.
4. Nginx forwards the API request to the Kubernetes backend Service.
5. The backend Service routes the request to the Node.js backend Pod.
6. The backend communicates with MongoDB Atlas.
7. The response travels back through the backend and Nginx to the browser.

---

## 3. Repository Structure

The DevOps repository is organized as follows:

```text
Devops_Proj1/
│
├── .github/
│   └── workflows/
│       └── build-images.yml
│
├── docker/
│   ├── backend/
│   │   ├── Dockerfile
│   │   └── .dockerignore
│   │
│   └── frontend/
│       ├── Dockerfile
│       ├── nginx.conf
│       └── .dockerignore
│
├── kubernetes/
│   ├── backend-deployment.yaml
│   ├── backend-service.yaml
│   ├── frontend-deployment.yaml
│   └── frontend-service.yaml
│
├── helm/
│
├── terraform/
│
└── README.md
```

The application repository is maintained separately:

```text
ElectronicsWeb/
│
├── client/
│   ├── src/
│   ├── package.json
│   └── package-lock.json
│
├── server/
│   ├── config/
│   ├── routes/
│   ├── models/
│   ├── package.json
│   └── index.js
│
├── package.json
└── README.md
```

The Docker configurations are maintained in the DevOps repository and copied into the appropriate application build contexts during the GitHub Actions workflow.

---

# 4. Docker Implementation

## 4.1 Backend Docker Configuration

The backend is a Node.js Express application running on port 5000.

The backend Dockerfile is maintained at:

```text
docker/backend/Dockerfile
```

The container runs the Node.js application using:

```bash
npm start
```

The backend requires the following environment variables:

| Variable | Description |
|---|---|
| MONGO_URI | MongoDB Atlas connection string |
| JWT_SECRET | Secret used for JWT authentication |
| PORT | Backend application port |

Sensitive configuration values are injected at runtime rather than committed to the repository.

## 4.2 Frontend Docker Configuration

The frontend uses React.js and Vite.

Docker uses a multi-stage build to reduce the final image size and avoid including Node.js development dependencies in the runtime container.

Dockerfile location:

```text
docker/frontend/Dockerfile
```

### Stage 1 – Build React Application

```dockerfile
FROM node:24-alpine AS builder

WORKDIR /app

COPY package*.json ./

RUN npm ci

COPY . .

ARG VITE_API_URL=/api
ENV VITE_API_URL=${VITE_API_URL}

RUN npm run build
```

This stage:

1. Uses Node.js Alpine as the build environment.
2. Copies package files.
3. Installs dependencies using `npm ci`.
4. Copies the application source code.
5. Sets the frontend API URL.
6. Builds the React application into the `dist` directory.

### Stage 2 – Serve Application Using Nginx

```dockerfile
FROM nginx:alpine

WORKDIR /app

COPY nginx.conf /etc/nginx/conf.d/default.conf

COPY --from=builder /app/dist /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

This stage uses Nginx to serve the production React build.

### Benefits of Multi-Stage Docker Builds

- Separates build and runtime environments.
- Avoids including unnecessary build dependencies in the final image.
- Reduces the final runtime image size.
- Improves container efficiency.

## 4.3 Nginx Reverse Proxy Configuration

File:

```text
docker/frontend/nginx.conf
```

Nginx is configured to serve the React application and forward API requests to the Kubernetes backend Service.

```nginx
server {
    listen 80;
    server_name _;

    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location /api/ {
        proxy_pass http://electronicsweb-backend-service:5000;
        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Explanation

**React Router Support**

```nginx
location / {
    try_files $uri $uri/ /index.html;
}
```

This ensures that React Router routes are handled correctly by returning `index.html` when a requested frontend route does not correspond to a physical file.

**API Reverse Proxy**

```nginx
location /api/ {
    proxy_pass http://electronicsweb-backend-service:5000;
}
```

This forwards API requests to the backend Kubernetes Service.

The Kubernetes Service provides a stable DNS name even when backend Pods are recreated.

The frontend does not need to know the backend Pod IP address.

**Important:** The Nginx configuration is updated to use the Kubernetes Service name instead of the Docker Compose service name.

---

# 5. CI Pipeline – GitHub Actions

## 5.1 Overview

GitHub Actions is used to automate Docker image creation and publishing.

Workflow file:

```text
.github/workflows/build-images.yml
```

The workflow is triggered when changes are pushed to the `main` branch affecting:

- `docker/**`
- `.github/workflows/**`

It can also be triggered manually using `workflow_dispatch`.

## 5.2 Workflow Configuration

The workflow performs the following operations:

### Step 1 – Checkout DevOps Repository

```yaml
- name: Checkout DevOps Repository
  uses: actions/checkout@v4
```

Checks out the repository containing the Docker configurations and GitHub Actions workflow.

### Step 2 – Checkout Application Repository

```yaml
- name: Checkout Application Repository
  uses: actions/checkout@v4
  with:
    repository: mkg789/ElectronicsWeb
    ref: main
    path: ElectronicsWeb
```

Checks out the ElectronicsWeb application source code into the workflow environment.

### Step 3 – Prepare Docker Build Contexts

```yaml
- name: Prepare Docker Build Contexts
  run: |
    cp docker/backend/.dockerignore ElectronicsWeb/server/.dockerignore
    cp docker/frontend/.dockerignore ElectronicsWeb/client/.dockerignore
    cp docker/frontend/nginx.conf ElectronicsWeb/client/nginx.conf
```

This step copies the Docker-related configuration into the appropriate application directories.

The frontend Nginx configuration is copied into the frontend Docker build context.

### Step 4 – Set Up Docker Buildx

```yaml
- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v3
```

Initializes Docker Buildx for building container images.

### Step 5 – Authenticate with GHCR

```yaml
- name: Login to GitHub Container Registry
  uses: docker/login-action@v3
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}
```

Authenticates the GitHub Actions workflow with GitHub Container Registry.

### Step 6 – Build and Push Backend Image

```yaml
- name: Build and Push Backend Image
  uses: docker/build-push-action@v6
  with:
    context: ./ElectronicsWeb/server
    file: ./docker/backend/Dockerfile
    tags: |
      ghcr.io/mkg789/electronicsweb-backend:latest
      ghcr.io/mkg789/electronicsweb-backend:${{ github.sha }}
    push: true
```

Builds the backend image and pushes it to GHCR.

Two tags are generated:

- `latest`
- Git commit SHA

### Step 7 – Build and Push Frontend Image

```yaml
- name: Build and Push Frontend Image
  uses: docker/build-push-action@v6
  with:
    context: ./ElectronicsWeb/client
    file: ./docker/frontend/Dockerfile
    build-args: |
      VITE_API_URL=/api
    tags: |
      ghcr.io/mkg789/electronicsweb-frontend:latest
      ghcr.io/mkg789/electronicsweb-frontend:${{ github.sha }}
    push: true
```

Builds the React frontend and publishes the image to GHCR.

The frontend is built with:

```text
VITE_API_URL=/api
```

## 5.3 Container Images

The GitHub Actions workflow publishes:

| Application | Image |
|---|---|
| Backend | `ghcr.io/mkg789/electronicsweb-backend:latest` |
| Frontend | `ghcr.io/mkg789/electronicsweb-frontend:latest` |

Commit-specific tags are also generated.

## 5.4 Current CI Pipeline Scope

The current workflow builds and pushes images to GHCR.

The transfer from GHCR to the local Harbor registry is performed manually.

The pipeline can be extended later to automate this transfer and deploy updated images through Argo CD.

---

# 6. Harbor Private Registry Setup

## 6.1 Overview

Harbor is used as the private container registry for storing application images.

Harbor was installed locally inside Ubuntu running under WSL2.

Version:

```text
Harbor v2.15.4
```

Registry configuration:

| Setting | Value |
|---|---|
| Registry hostname | harbor.local |
| HTTP port | 80 |
| Project | electronicsweb |
| Backend repository | electronicsweb-backend |
| Frontend repository | electronicsweb-frontend |

A private Harbor project named `electronicsweb` was created.

Both backend and frontend images were transferred from GHCR into Harbor.

## 6.2 GHCR to Harbor Image Transfer

The image transfer involves three operations.

### Pull Image from GHCR

```bash
docker pull ghcr.io/mkg789/electronicsweb-frontend:latest
```

### Tag Image for Harbor

```bash
docker tag ghcr.io/mkg789/electronicsweb-frontend:latest \
harbor.local:80/electronicsweb/electronicsweb-frontend:latest
```

### Push Image to Harbor

```bash
docker login harbor.local:80
```

```bash
docker push harbor.local:80/electronicsweb/electronicsweb-frontend:latest
```

The backend image was transferred using the same process.

### Images Stored in Harbor

```text
harbor.local:80/electronicsweb/electronicsweb-backend:latest

harbor.local:80/electronicsweb/electronicsweb-frontend:latest
```

## 6.3 Harbor Robot Account

A Harbor robot account named `k8s-pull` was created for Kubernetes image-pull authentication.

Permission:

```text
Repository: Pull
```

The robot account is used by Kubernetes to authenticate when pulling private images.

The generated robot secret is stored securely and is not committed to GitHub.

## 6.4 HTTP Registry Configuration

For this isolated local learning environment, Harbor was configured to use HTTP.

The Docker daemon was configured to recognize the local registry as insecure.

Minikube runs containerd, so containerd required a separate registry configuration.

The Minikube node uses the host gateway address:

```text
192.168.49.1:80
```

This address is used inside Kubernetes because `harbor.local` resolves differently inside the Minikube node.

Containerd configuration:

```text
/etc/containerd/certs.d/192.168.49.1:80/hosts.toml
```

Configuration:

```toml
server = "http://192.168.49.1:80"

[host."http://192.168.49.1:80"]
  capabilities = ["pull", "resolve"]
```

Containerd uses this configuration to communicate with Harbor over HTTP.

**Security note:** HTTP is used only for this isolated local lab. Production registries should use HTTPS with valid TLS certificates. Do not expose this HTTP registry publicly.

---

# 7. Kubernetes Environment Setup

## 7.1 Local Kubernetes Cluster

Minikube is used to run a local Kubernetes cluster.

Configuration:

| Component | Configuration |
|---|---|
| Kubernetes distribution | Minikube |
| Driver | Docker |
| Kubernetes version | v1.37.0 |
| Container runtime | containerd |
| Node | minikube |
| Namespace | electronicsweb |

Cluster initialization:

```bash
minikube start --driver=docker
```

Verify cluster status:

```bash
minikube status
```

Check nodes:

```bash
kubectl get nodes -o wide
```

Expected node status:

```text
NAME       STATUS   ROLES           VERSION
minikube   Ready    control-plane   v1.37.0
```

## 7.2 Kubernetes Namespace

A dedicated namespace named `electronicsweb` was created to isolate the application resources.

```bash
kubectl create namespace electronicsweb
```

Verify:

```bash
kubectl get namespaces
```

All application Deployments, Services and Secrets are maintained in this namespace.

---

# 8. Kubernetes Private Registry Authentication

Kubernetes requires authentication to pull images from the private Harbor project.

## 8.1 Create Image Pull Secret

A Docker registry secret was created in the `electronicsweb` namespace.

```bash
kubectl create secret docker-registry harbor-registry \
  --namespace electronicsweb \
  --docker-server=192.168.49.1:80 \
  --docker-username='<HARBOR_ROBOT_USERNAME>' \
  --docker-password='<HARBOR_ROBOT_SECRET>'
```

The actual credentials must be supplied locally and must never be committed to Git.

The resulting secret type is:

```text
kubernetes.io/dockerconfigjson
```

Verify:

```bash
kubectl get secret harbor-registry -n electronicsweb
```

## 8.2 Attach Secret to Default ServiceAccount

The Harbor image-pull secret was attached to the default ServiceAccount.

```bash
kubectl patch serviceaccount default \
  -n electronicsweb \
  -p '{"imagePullSecrets":[{"name":"harbor-registry"}]}'
```

Verify:

```bash
kubectl get serviceaccount default -n electronicsweb -o yaml
```

Expected configuration:

```yaml
imagePullSecrets:
  - name: harbor-registry
```

This allows Pods using the default ServiceAccount to authenticate with Harbor.

## 8.3 Image Pull Verification

A temporary Pod was created to verify that Kubernetes could pull the private backend image.

```bash
kubectl run harbor-pull-test \
  -n electronicsweb \
  --image=192.168.49.1:80/electronicsweb/electronicsweb-backend:latest \
  --restart=Never \
  --command -- sleep 300
```

The Pod reached:

```text
STATUS: Running
READY: 1/1
```

This confirmed:

- Harbor connectivity from Minikube.
- Containerd HTTP registry configuration.
- Kubernetes image-pull secret authentication.
- Successful private image download.

The temporary test Pod was subsequently deleted.

---

# 9. Kubernetes Backend Deployment

## 9.1 Backend Configuration

The backend application runs on port 5000.

Its database connection is configured using:

```javascript
mongoose.connect(process.env.MONGO_URI);
```

The backend also requires:

- `MONGO_URI`
- `JWT_SECRET`
- `PORT`

Sensitive values are stored in Kubernetes Secrets.

## 9.2 Create Backend Secret

```bash
kubectl create secret generic electronicsweb-backend-secrets \
  -n electronicsweb \
  --from-literal=MONGO_URI='<MONGODB_URI>' \
  --from-literal=JWT_SECRET='<JWT_SECRET>'
```

Actual secret values are not included in the repository.

Verify:

```bash
kubectl get secret electronicsweb-backend-secrets -n electronicsweb
```

Expected:

```text
NAME                            TYPE     DATA
electronicsweb-backend-secrets  Opaque   2
```

## 9.3 Backend Deployment

File:

```text
kubernetes/backend-deployment.yaml
```

Configuration:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: electronicsweb-backend
  namespace: electronicsweb
spec:
  replicas: 1
  selector:
    matchLabels:
      app: electronicsweb-backend
  template:
    metadata:
      labels:
        app: electronicsweb-backend
    spec:
      imagePullSecrets:
        - name: harbor-registry
      containers:
        - name: backend
          image: 192.168.49.1:80/electronicsweb/electronicsweb-backend:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 5000
          env:
            - name: PORT
              value: "5000"
            - name: MONGO_URI
              valueFrom:
                secretKeyRef:
                  name: electronicsweb-backend-secrets
                  key: MONGO_URI
            - name: JWT_SECRET
              valueFrom:
                secretKeyRef:
                  name: electronicsweb-backend-secrets
                  key: JWT_SECRET
```

Apply:

```bash
kubectl apply -f kubernetes/backend-deployment.yaml
```

Verify:

```bash
kubectl get deployments -n electronicsweb

kubectl get pods -n electronicsweb
```

## 9.4 Backend Logs

Backend application logs were verified using:

```bash
kubectl logs deployment/electronicsweb-backend -n electronicsweb
```

Successful output:

```text
> server@1.0.0 start
> node index.js

Server running on port 5000
MongoDB Connected
```

This confirmed that:

- The backend container started successfully.
- Express was listening on port 5000.
- MongoDB Atlas connectivity was established.

---

# 10. Kubernetes Backend Service

## 10.1 Why a Service Is Required

Pod IP addresses are temporary and can change when Pods are recreated.

A Kubernetes Service provides a stable network endpoint for accessing backend Pods.

A ClusterIP Service was created for internal communication.

File:

```text
kubernetes/backend-service.yaml
```

Configuration:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: electronicsweb-backend-service
  namespace: electronicsweb
spec:
  type: ClusterIP
  selector:
    app: electronicsweb-backend
  ports:
    - protocol: TCP
      port: 5000
      targetPort: 5000
```

Apply:

```bash
kubectl apply -f kubernetes/backend-service.yaml
```

Verify:

```bash
kubectl get services -n electronicsweb
```

The Service is available internally using:

```text
electronicsweb-backend-service:5000
```

The full namespace-qualified DNS name is:

```text
electronicsweb-backend-service.electronicsweb.svc.cluster.local
```

## 10.2 EndpointSlice Verification

```bash
kubectl get endpointslice -n electronicsweb
```

The EndpointSlice showed the backend Pod IP and port 5000.

This verified that the Service selector successfully identified the backend Pod.

---

# 11. Kubernetes Frontend Deployment

## 11.1 Frontend Image

The frontend image is stored in Harbor:

```text
192.168.49.1:80/electronicsweb/electronicsweb-frontend:latest
```

The frontend container serves the React application using Nginx on port 80.

## 11.2 Frontend Deployment

File:

```text
kubernetes/frontend-deployment.yaml
```

Configuration:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: electronicsweb-frontend
  namespace: electronicsweb
spec:
  replicas: 1
  selector:
    matchLabels:
      app: electronicsweb-frontend
  template:
    metadata:
      labels:
        app: electronicsweb-frontend
    spec:
      imagePullSecrets:
        - name: harbor-registry
      containers:
        - name: frontend
          image: 192.168.49.1:80/electronicsweb/electronicsweb-frontend:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 80
```

Apply:

```bash
kubectl apply -f kubernetes/frontend-deployment.yaml
```

Verify:

```bash
kubectl get deployments -n electronicsweb

kubectl get pods -n electronicsweb
```

Both frontend and backend Pods reached the Running state.

---

# 12. Kubernetes Frontend Service

## 12.1 Why NodePort?

The frontend must be accessible from outside the Kubernetes cluster.

A NodePort Service exposes the frontend using a port on the Minikube node.

File:

```text
kubernetes/frontend-service.yaml
```

Configuration:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: electronicsweb-frontend-service
  namespace: electronicsweb
spec:
  type: NodePort
  selector:
    app: electronicsweb-frontend
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30080
```

Apply:

```bash
kubectl apply -f kubernetes/frontend-service.yaml
```

Verify:

```bash
kubectl get services -n electronicsweb
```

The frontend is exposed using:

```text
NodePort: 30080
```

## 12.2 Access Application

Run:

```bash
minikube service electronicsweb-frontend-service \
  -n electronicsweb \
  --url
```

Minikube returns a URL that can be opened in a browser.

The React application was successfully accessed through this URL.

---

# 13. Final Deployment Verification

The following commands can be used to verify the application.

### Check Kubernetes Nodes

```bash
kubectl get nodes
```

### Check Deployments

```bash
kubectl get deployments -n electronicsweb
```

### Check Pods

```bash
kubectl get pods -n electronicsweb
```

### Check Services

```bash
kubectl get services -n electronicsweb
```

### Check Backend Logs

```bash
kubectl logs deployment/electronicsweb-backend \
  -n electronicsweb
```

### Check Frontend Logs

```bash
kubectl logs deployment/electronicsweb-frontend \
  -n electronicsweb
```

### Check EndpointSlices

```bash
kubectl get endpointslice -n electronicsweb
```

### Final Application Status

| Component | Configuration | Status |
|---|---|---|
| Harbor | Private local registry | Running |
| Minikube | Kubernetes v1.37.0 | Running |
| Namespace | electronicsweb | Created |
| Backend Deployment | 1 replica | Running |
| Backend Service | ClusterIP :5000 | Configured |
| Frontend Deployment | 1 replica | Running |
| Frontend Service | NodePort :30080 | Configured |
| MongoDB Atlas | External database | Connected |
| Harbor Image Authentication | Kubernetes Secret | Verified |
| Frontend-to-Backend Communication | Nginx reverse proxy | Working |
| Web Application | React + Node.js | Accessible |

---

# 14. Troubleshooting and Issues Resolved

During implementation, several practical issues were encountered and resolved.

## 14.1 Harbor Registry HTTP/HTTPS Issue

**Problem:**

Docker attempted to communicate with Harbor using HTTPS, while Harbor was configured for HTTP.

**Resolution:**

- Configured the Docker daemon to recognize the local insecure registry.
- Used the explicit registry port `80`.
- Logged in using the exact registry address:

```bash
docker login harbor.local:80
```

## 14.2 Harbor Authentication Failure

**Problem:**

Docker returned:

```text
no basic auth credentials
```

**Cause:**

The Docker login address and image registry address did not match.

**Resolution:**

Logged into the exact registry endpoint:

```bash
docker login harbor.local:80
```

## 14.3 Minikube Could Not Resolve harbor.local

**Problem:**

Inside Minikube, `harbor.local` resolved to `127.0.0.1`, which pointed to the Minikube node itself.

**Resolution:**

Used the Minikube host gateway:

```text
192.168.49.1:80
```

Configured containerd to communicate with Harbor using this address.

## 14.4 Containerd HTTP Registry Configuration

**Problem:**

Containerd attempted to use HTTPS for the Harbor registry.

**Resolution:**

Created:

```text
/etc/containerd/certs.d/192.168.49.1:80/hosts.toml
```

Configured the registry to use HTTP.

## 14.5 Kubernetes Image Pull Authentication

**Problem:**

The initial test using `crictl pull` failed with:

```text
no basic auth credentials
```

**Cause:**

`crictl` does not automatically use Kubernetes Pod image-pull Secrets.

**Resolution:**

Created a Kubernetes Docker registry Secret and attached it to the default ServiceAccount.

Verified authentication using an actual Kubernetes Pod.

## 14.6 Harbor Robot Username Incorrectly Stored

**Problem:**

The robot username was stored in Kubernetes as:

```text
robot+k8s-pull
```

Instead of:

```text
robot$electronicsweb+k8s-pull
```

**Cause:**

Bash interpreted `$electronicsweb` as a shell variable because the username was enclosed in double quotes.

**Resolution:**

Used single quotes to preserve the literal dollar sign:

```bash
--docker-username='robot$electronicsweb+k8s-pull'
```

Recreated the Kubernetes Secret and successfully pulled the image.

## 14.7 Backend MongoDB Connection Error

**Problem:**

The backend container initially failed with:

```text
MongoParseError: Invalid scheme, expected connection string
to start with "mongodb://" or "mongodb+srv://"
```

**Cause:**

The `MONGO_URI` value supplied through the Kubernetes Secret was not a valid MongoDB connection string.

**Resolution:**

Corrected the stored configuration and restarted the Deployment.

Verified successful application startup:

```text
Server running on port 5000
MongoDB Connected
```

## 14.8 Frontend Nginx Backend Connectivity

**Problem:**

The original Nginx configuration used:

```nginx
proxy_pass http://backend:5000;
```

This referred to the Docker Compose service name.

**Resolution:**

Updated the Nginx configuration to use the Kubernetes Service:

```nginx
proxy_pass http://electronicsweb-backend-service:5000;
```

Rebuilt the frontend image through GitHub Actions, transferred the updated image to Harbor and deployed it in Kubernetes.

---

# 15. Starting and Stopping the Local DevOps Environment

The project runs locally using WSL2, Docker Desktop, Harbor and Minikube.

To conserve laptop resources, the environment can be stopped when not in use.

## 15.1 Start the Environment

Start Ubuntu/WSL and run:

```bash
minikube start
```

Start Harbor:

```bash
cd ~/harbor-lab/harbor

sudo docker compose up -d
```

Verify Harbor:

```bash
docker compose ps
```

Verify Kubernetes:

```bash
minikube status

kubectl get nodes
```

## 15.2 Stop Harbor

```bash
cd ~/harbor-lab/harbor

sudo docker compose down
```

This stops Harbor containers while preserving the stored data and images.

Do not use `docker compose down -v` because it removes volumes.

## 15.3 Stop Minikube

```bash
minikube stop
```

## 15.4 Shut Down WSL

From Windows PowerShell:

```powershell
wsl --shutdown
```

The Harbor data and Minikube cluster are retained for subsequent sessions.

---

# 16. Security Considerations

The project follows basic secret-management practices.

- MongoDB connection strings are stored in Kubernetes Secrets.
- JWT signing secrets are stored in Kubernetes Secrets.
- Harbor robot credentials are stored in a Kubernetes image-pull Secret.
- Sensitive credentials are not included in application manifests.
- Credentials must not be committed to GitHub.
- HTTP Harbor configuration is restricted to the local learning environment.

**Important production considerations:**

Kubernetes Secret values are base64-encoded by default; base64 is not encryption.

For production deployments, additional security controls should be implemented:

- Kubernetes encryption at rest.
- Restricted RBAC permissions.
- External secret-management solutions such as HashiCorp Vault or cloud secret managers.
- HTTPS-enabled Harbor registry.
- Non-root containers where supported.
- Resource requests and limits.
- Network Policies.
- Image vulnerability scanning.
- Immutable image tags instead of relying only on `latest`.

---

# 17. Future Enhancements / Planned Work

The current implementation successfully deploys the full-stack application on Minikube.

The following improvements are planned.

## 17.1 Helm Charts

Convert the existing Kubernetes YAML manifests into reusable Helm charts.

Expected structure:

```text
helm/
└── electronicsweb/
    ├── Chart.yaml
    ├── values.yaml
    └── templates/
        ├── backend-deployment.yaml
        ├── backend-service.yaml
        ├── frontend-deployment.yaml
        └── frontend-service.yaml
```

Benefits:

- Reusable deployment templates.
- Centralized configuration.
- Easier application upgrades.
- Environment-specific values.
- Simplified release management.

## 17.2 Argo CD – GitOps

Integrate Argo CD to implement GitOps-based deployments.

Expected workflow:

```text
Developer
    |
    v
GitHub
    |
    v
GitHub Actions
    |
    v
Build Docker Images
    |
    v
GHCR
    |
    v
Harbor
    |
    v
Update Deployment Configuration in Git
    |
    v
Argo CD
    |
    v
Kubernetes
    |
    v
Application Deployment
```

Argo CD will monitor the Git repository and synchronize Kubernetes resources with the declared configuration.

## 17.3 Monitoring

Introduce:

- Prometheus for metrics collection.
- Grafana for monitoring dashboards.
- Kubernetes resource monitoring.
- Application health monitoring.
- Container logs and troubleshooting.

## 17.4 Improved CI/CD

Extend GitHub Actions to automate:

- Frontend and backend image builds.
- Image tagging.
- Image security scanning.
- Publishing to Harbor.
- Updating deployment manifests or Helm values.
- Triggering Argo CD synchronization.

## 17.5 Production-Ready Kubernetes Configuration

Additional improvements:

- Liveness probes.
- Readiness probes.
- CPU and memory requests and limits.
- Horizontal Pod Autoscaling.
- Ingress configuration.
- TLS certificates.
- Persistent storage where required.
- Multiple replicas and rolling updates.

---

# 18. Conclusion

This project demonstrates the practical implementation of a DevOps workflow for a full-stack application.

The implementation covers:

- Docker multi-stage builds.
- GitHub Actions CI automation.
- Container image publishing using GHCR.
- Private container registry management using Harbor.
- Kubernetes cluster configuration using Minikube.
- Private registry authentication.
- Kubernetes Deployments and Services.
- Application configuration using Kubernetes Secrets.
- Frontend and backend communication through Nginx.
- Deployment verification and troubleshooting.

The ElectronicsWeb application is successfully running on the local Kubernetes cluster, with the React frontend communicating with the Node.js backend and MongoDB Atlas.

The next phase will focus on Helm-based deployments and Argo CD GitOps automation.

---

**Project Status:** Phase 1 – Containerization, CI and Kubernetes Deployment Completed.

**Environment:** Local DevOps Lab – Windows, WSL2, Docker Desktop and Minikube.

