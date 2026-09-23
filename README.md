# StreamingApp — Container Orchestration & Scaling on AWS EKS

A **5-service MERN streaming platform** containerized with Docker, packaged with Helm, and deployed to **Amazon EKS**.

The project demonstrates containerization, Kubernetes orchestration, Ingress-based routing, horizontal scaling, rolling updates, self-healing, CI/CD with Jenkins, external MongoDB Atlas integration, S3-backed media storage, and CloudWatch monitoring/logging.

---

## 1. Project Overview

StreamingApp is a microservices-based streaming application consisting of:

- React frontend
- Authentication service
- Streaming service
- Admin service
- Real-time chat service

The application is deployed on an **Amazon EKS Kubernetes cluster** and uses AWS-managed services for storage, monitoring, and notifications.

### Key Technologies

| Category | Technology |
|---|---|
| Frontend | React, Nginx |
| Backend | Node.js, Express.js |
| Database | MongoDB Atlas |
| Containerization | Docker |
| Orchestration | Kubernetes |
| Kubernetes Packaging | Helm 3 |
| Cloud | AWS |
| Kubernetes Platform | Amazon EKS |
| Media Storage | Amazon S3 |
| Monitoring | Amazon CloudWatch |
| Notifications | Amazon SNS |
| CI/CD | Jenkins |
| Real-time Communication | Socket.IO |
| Container Registry | Docker Hub |

---

# 2. Architecture

```text
                              ┌─────────────────────────┐
                              │     NGINX INGRESS       │
                              │   streamingapp.local    │
                              └────────────┬────────────┘
                                           │
              ┌────────────────────────────┼────────────────────────────┐
              │                            │                            │
              ▼                            ▼                            ▼
       ┌─────────────┐             ┌─────────────┐              ┌─────────────┐
       │  Frontend   │             │  Auth API   │              │ Streaming   │
       │   React     │             │    :3001    │              │    :3002    │
       └─────────────┘             └─────────────┘              └─────────────┘
                                           │                            │
                                           │                            │
                                           ▼                            ▼
                                    ┌─────────────┐              ┌─────────────┐
                                    │   Admin     │              │    Chat     │
                                    │    :3003    │              │ :3004 + WS  │
                                    └─────────────┘              └─────────────┘
                                           │                            │
                                           └────────────┬───────────────┘
                                                        │
                         ┌──────────────────────────────┼─────────────────────────────┐
                         │                              │                             │
                         ▼                              ▼                             ▼
                  ┌──────────────┐              ┌──────────────┐              ┌──────────────┐
                  │ MongoDB Atlas│              │  Amazon S3   │              │ CloudWatch   │
                  │   Database   │              │Video/Thumbs  │              │ Logs/Metrics │
                  └──────────────┘              └──────────────┘              └──────────────┘
```

### Request Routing

| Request | Destination |
|---|---|
| `/` | Frontend |
| `/api/auth/*` | Auth service |
| `/api/streaming/*` | Streaming service |
| `/api/admin/*` | Admin service |
| `/api/chat/*` | Chat service |
| `/socket.io/*` | Chat service |

### Service Routing Details

- **Frontend** — React SPA served through Nginx.
- **Auth Service** — Express router mounted at `/api`. Ingress rewrites `/api/auth/*` to `/api/*`.
- **Streaming Service** — Express router mounted at `/api/streaming`.
- **Admin Service** — Express router mounted at `/api/admin`.
- **Chat Service** — Express router mounted at `/api/chat`.
- **Socket.IO** — Uses the default `/socket.io` path and is routed directly to the chat service with extended proxy timeouts for persistent WebSocket connections.

---

# 3. AWS & External Services

### MongoDB Atlas

MongoDB is hosted externally using **MongoDB Atlas** rather than running MongoDB inside the Kubernetes cluster.

This avoids the additional complexity of:

- Kubernetes StatefulSets
- Persistent Volumes
- EBS CSI configuration
- StorageClasses
- Database backup management inside EKS

### Amazon S3

Video and thumbnail files are stored in Amazon S3.

The application uses **presigned S3 URLs** for media upload operations.

```text
Admin / Streaming Service
          │
          ▼
   Generate Presigned URL
          │
          ▼
        Client
          │
          ▼
      Amazon S3
```

### Amazon CloudWatch

CloudWatch is used for:

- Container logs
- Node logs
- Application monitoring
- Kubernetes metrics
- CPU monitoring
- Alarm notifications

---

# 4. Repository Structure

```text
streamingapp/
│
├── backend/
│   ├── authService/
│   ├── streamingService/
│   ├── adminService/
│   └── chatService/
│
├── frontend/
│   ├── src/
│   ├── Dockerfile
│   └── nginx.conf
│
├── helm-chart/
│   ├── Chart.yaml
│   ├── values.yaml
│   │
│   └── templates/
│       ├── auth-deployment.yaml
│       ├── auth-service.yaml
│       ├── streaming-deployment.yaml
│       ├── streaming-service.yaml
│       ├── admin-deployment.yaml
│       ├── admin-service.yaml
│       ├── chat-deployment.yaml
│       ├── chat-service.yaml
│       ├── frontend-deployment.yaml
│       ├── frontend-service.yaml
│       ├── configmap.yaml
│       ├── secret.yaml
│       └── ingress.yaml
│
└── Jenkinsfile
```

---

# 5. Prerequisites

Install and configure:

- Docker Desktop
- AWS CLI
- kubectl
- Helm 3
- eksctl
- Jenkins
- AWS account
- MongoDB Atlas account
- Docker Hub account

Verify installations:

```powershell
docker --version
aws --version
kubectl version --client
helm version
eksctl version
```

Configure AWS:

```powershell
aws configure
```

Verify the AWS identity:

```powershell
aws sts get-caller-identity
```

---

# 6. Create the EKS Cluster

Create the Kubernetes cluster:

```powershell
eksctl create cluster `
  --name streamingapp-cluster `
  --region us-east-1 `
  --nodes 2 `
  --node-type t3.medium
```

Verify:

```powershell
kubectl get nodes
```

Expected:

```text
NAME                                  STATUS   ROLES
ip-xxx-xxx-xxx-xxx.ec2.internal      Ready    <none>
ip-xxx-xxx-xxx-xxx.ec2.internal      Ready    <none>
```

---

# 7. Install Nginx Ingress Controller

Add the Helm repository:

```powershell
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

Create the namespace:

```powershell
kubectl create namespace ingress-nginx
```

Install the controller:

```powershell
helm install ingress-nginx `
  ingress-nginx/ingress-nginx `
  --namespace ingress-nginx
```

Verify:

```powershell
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

The Ingress controller exposes the application through an AWS LoadBalancer.

---

# 8. Configure Application Secrets

The application requires:

- JWT secret
- AWS access key
- AWS secret key
- MongoDB Atlas connection string

Example:

```powershell
helm install streamingapp ./helm-chart `
  --set secrets.jwtSecret="<your-jwt-secret>" `
  --set secrets.awsAccessKeyId="<your-aws-access-key>" `
  --set secrets.awsSecretAccessKey="<your-aws-secret-key>" `
  --set "secrets.mongoUri=mongodb+srv://<username>:<password>@<cluster>.mongodb.net/streamingapp?retryWrites=true&w=majority"
```

> **Security:** Never commit real credentials, AWS keys, MongoDB passwords, or JWT secrets to Git.

---

# 9. Verify the Deployment

Check Kubernetes resources:

```powershell
kubectl get pods
kubectl get deployments
kubectl get services
kubectl get ingress
```

The deployment should contain the five application services:

```text
frontend
auth
streaming
admin
chat
```

Check Helm:

```powershell
helm list
```

Check the release:

```powershell
helm status streamingapp
```

---

# 10. Access the Application

Get the Ingress:

```powershell
kubectl get ingress
```

Or:

```powershell
kubectl get svc -n ingress-nginx
```

Resolve the AWS LoadBalancer hostname to an IP address and add an entry to:

```text
C:\Windows\System32\drivers\etc\hosts
```

Example:

```text
<resolved-ip> streamingapp.local
```

Then open:

```text
http://streamingapp.local
```

---

# 11. Updating the Application

When deploying a new image version:

```powershell
helm upgrade streamingapp ./helm-chart `
  --reuse-values `
  --set services.frontend.tag=<new-tag>
```

Check the rollout:

```powershell
kubectl rollout status deployment/frontend-deployment
```

### Important

Use `--reuse-values` when upgrading if the deployment depends on values supplied during the original installation.

Without it, previously supplied values may fall back to chart defaults.

---

# 12. Scaling

Scale a deployment manually:

```powershell
kubectl scale deployment/streaming-deployment --replicas=4
```

Verify:

```powershell
kubectl get pods
```

Check rollout:

```powershell
kubectl rollout status deployment/streaming-deployment
```

Scale back down:

```powershell
kubectl scale deployment/streaming-deployment --replicas=2
```

---

# 13. Rolling Updates

All application Deployments use Kubernetes `RollingUpdate`.

Configuration:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0
    maxSurge: 1
```

Deploy a new version:

```powershell
helm upgrade streamingapp ./helm-chart `
  --reuse-values `
  --set services.frontend.tag=<new-tag>
```

Monitor:

```powershell
kubectl rollout status deployment/frontend-deployment
```

Rollback if required:

```powershell
helm rollback streamingapp <revision>
```

---

# 14. Self-Healing

Kubernetes automatically recreates failed pods because the application is managed by Deployments.

Check pods:

```powershell
kubectl get pods
```

Delete a running pod:

```powershell
kubectl delete pod <pod-name>
```

Watch Kubernetes recreate it:

```powershell
kubectl get pods -w
```

The Deployment controller automatically creates a replacement pod to maintain the desired replica count.

---

# 15. Jenkins CI Pipeline

The project includes a `Jenkinsfile` for automating Docker image builds and publishing.

Typical pipeline:

```text
Git Push
   │
   ▼
Jenkins
   │
   ├── Checkout
   ├── Build Frontend
   ├── Build Backend Services
   ├── Build Docker Images
   ├── Tag Images
   └── Push Images
           │
           ▼
       Docker Hub
```

The Jenkins pipeline automates:

- Source checkout
- Docker image builds
- Image tagging
- Docker Hub authentication
- Image publishing

Verify an image:

```powershell
docker pull <dockerhub-user>/<image>:<tag>
```

---

# 16. Monitoring & Logging

Amazon CloudWatch Observability is enabled using the EKS add-on:

```text
amazon-cloudwatch-observability
```

CloudWatch collects:

- Container logs
- Application logs
- Node logs
- Kubernetes metrics
- CPU utilization
- Performance information

Relevant log groups:

```text
/aws/containerinsights/streamingapp-cluster/*
```

Container Insights provides visibility into:

```text
Application
Dataplane
Host
Performance
```

---

# 17. CloudWatch Alarm

A CPU alarm was configured:

```text
Alarm:
streamingapp-high-cpu

Condition:
Node CPU > 80%

Evaluation:
2 evaluation periods

Notification:
SNS → Email
```

SNS topic:

```text
streamingapp-alerts
```

The email subscription must be confirmed before notifications are delivered.

---

# 18. IAM Requirements

The CloudWatch agent requires appropriate AWS permissions.

The EKS node group IAM role must have:

```text
CloudWatchAgentServerPolicy
```

attached.

Without the required permissions, CloudWatch agent / Fluent Bit components may fail to authenticate and publish logs or metrics.

---

# 19. Chat & WebSocket Support

The Chat service uses Socket.IO for real-time communication.

```text
Browser
   │
   │ Socket.IO
   ▼
Nginx Ingress
   │
   ▼
chat-service
   │
   ▼
Chat API
```

The Ingress configuration includes extended proxy timeouts for long-lived WebSocket connections.

Live chat was verified between two browser tabs.

---

# 20. S3 Media Upload

The admin application supports video and thumbnail uploads using Amazon S3.

Upload flow:

```text
Admin UI
   │
   ▼
Backend Service
   │
   │ Generate presigned URL
   ▼
Browser
   │
   ▼
Amazon S3
```

Required S3 permissions include:

```text
s3:PutObject
s3:GetObject
s3:DeleteObject
```

The Ingress request body size must also be increased because the default Nginx limit is insufficient for larger video uploads.

---

# 21. Important Deployment Issues Resolved

### EKS Node Creation Failure

The initial node group experienced a `NodeCreationFailure`.

The cluster/node configuration was recreated and validated before continuing with the application deployment.

### Ingress Rewrite Mismatch

The backend services use different Express router prefixes:

```text
Auth       → /api
Streaming  → /api/streaming
Admin      → /api/admin
Chat       → /api/chat
```

Therefore, separate Ingress rules were configured instead of using one generic rewrite rule.

### Frontend Environment Variable Issue

The frontend intentionally uses an empty value to represent same-origin API access.

A fallback incorrectly converted the empty value into:

```text
localhost
```

This caused API, video, and chat requests to target the wrong host.

The fallback logic was corrected so the application works through the Kubernetes Ingress hostname.

### React SPA Routing

Nginx required SPA fallback routing:

```nginx
try_files $uri $uri/ /index.html;
```

The Nginx configuration also had to be copied into the correct Docker build stage.

### S3 Upload Permissions

Media uploads initially failed because the required S3 permissions were not available.

The IAM policy was updated to allow the required object operations.

Nginx request body size was also increased to support larger video uploads.

### CloudWatch IAM

The CloudWatch agents initially required additional IAM permissions.

Attaching:

```text
CloudWatchAgentServerPolicy
```

to the EKS node group role allowed the observability components to authenticate successfully.

---

# 22. Verification Checklist

The following functionality was verified during deployment:

- [x] EKS cluster created successfully
- [x] Kubernetes nodes in `Ready` state
- [x] Docker images built successfully
- [x] Docker images pushed to Docker Hub
- [x] Helm chart installed successfully
- [x] All five application services deployed
- [x] Ingress routing working
- [x] Frontend accessible through `streamingapp.local`
- [x] User registration/login working
- [x] Video upload working
- [x] Video playback working
- [x] S3 media storage working
- [x] Live chat working across browser tabs
- [x] Kubernetes scaling verified
- [x] Rolling update verified
- [x] Pod self-healing verified
- [x] Jenkins pipeline verified
- [x] CloudWatch logs verified
- [x] CloudWatch metrics verified
- [x] SNS notification configuration verified

---

# 23. Screenshots

The following screenshots are included in the submission package.

| Screenshot | Description |
|---|---|
| `Cluster_state.png` | Kubernetes cluster resources using `kubectl get pods,svc,ingress -A` |
| `kubectl pods.png` | All five application services running |
| `pods running.png` | Pod health and replica confirmation |
| `docker push.png` | Docker image build and push |
| `Push success.png` | Successful Docker image push |
| `Version Check.png` | Image tag/version verification |
| `Jenkins success.png` | Successful Jenkins CI pipeline |
| `helm deployed.png` | Successful Helm installation |
| `helm upgrade.png` | Successful Helm upgrade |
| `successfully rolled out.png` | Rolling update completion |
| `scaling and update.png` | Live Kubernetes scaling |
| `scale down.png` | Scaling deployment back down |
| `pod self heal.png` | Automatically recreated pod |
| `streaming_app local.png` | Application running through `streamingapp.local` |
| `video upload.png` | Successful video and thumbnail upload |
| `video playback.png` | Video playback from the application |
| `sns created.png` | SNS topic and confirmed subscription |
| `cloudwatch logs.png` | CloudWatch logs and Container Insights |

Live chat across two browser tabs was also verified using the Socket.IO connection routed through the `/socket.io` Ingress path.

---

# 24. Production Improvements

The current deployment demonstrates a production-style Kubernetes architecture. For a real production environment, the following improvements are recommended.

### Security

- Enable HTTPS/TLS using cert-manager and AWS-managed certificates.
- Use a real domain instead of `/etc/hosts`.
- Avoid long-lived AWS access keys inside Kubernetes Secrets.
- Use IAM Roles for Service Accounts (IRSA) or EKS Pod Identity.
- Restrict MongoDB Atlas network access to the EKS NAT gateway or private network.
- Apply least-privilege S3 IAM policies.

### Kubernetes

- Replace manual `kubectl scale` commands with Horizontal Pod Autoscaler.
- Configure resource requests and limits for every workload.
- Add readiness and liveness probes.
- Use PodDisruptionBudgets for critical services.
- Separate environments into `dev`, `staging`, and `prod` namespaces.
- Maintain environment-specific Helm values.

### Deployment

- Add automated Helm deployments to Jenkins.
- Add automated rollback on failed deployments.
- Add image vulnerability scanning.
- Use immutable image tags rather than relying on mutable tags.

### Configuration

ConfigMap/Secret changes should automatically trigger rolling deployments.

A checksum annotation can be added to the Deployment template so configuration changes result in new ReplicaSets.

### Observability

For a production system, monitoring can be extended with:

- Application-level metrics
- Request latency
- Error rate
- HTTP status metrics
- Database metrics
- S3 metrics
- Distributed tracing
- Centralized alerting

---

# 25. Conclusion

This project demonstrates the complete container orchestration lifecycle of a MERN-based streaming application on AWS:

```text
Application
     │
     ▼
Docker
     │
     ▼
Docker Hub
     │
     ▼
Helm
     │
     ▼
Amazon EKS
     │
     ├── Ingress Routing
     ├── Scaling
     ├── Rolling Updates
     ├── Self-Healing
     └── WebSocket Support
     │
     ├── MongoDB Atlas
     ├── Amazon S3
     └── CloudWatch + SNS
```
---

## Project Status

**Status: Completed and verified**

The application was successfully deployed to Amazon EKS and the core application, scaling, deployment, storage, monitoring, logging, CI/CD, and self-healing scenarios were tested.
