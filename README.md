# StreamingApp on Kubernetes (EKS) – Orchestration and Scaling

A multi-service MERN streaming platform, containerized, built by Jenkins, stored in Amazon ECR and deployed to Amazon EKS with Helm, Ingress, CloudWatch monitoring and logging.

| Service | Port | Role |
|---|---|---|
| authService | 3001 | Registration, login, JWT |
| streamingService | 3002 | Video catalogue, playback |
| adminService | 3003 | Asset management, uploads |
| chatService | 3004 | WebSocket + REST chat |
| frontend | 80 | React SPA served by Nginx |
| MongoDB | 27017 | Shared database (StatefulSet) |

Full diagrams: [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)

## Repository layout

```
.
├── backend/                 # authService, streamingService, adminService, chatService
├── frontend/                # React app + Dockerfile (multi-stage: Node -> Nginx)
├── streamingapp/            # Helm chart (Chart.yaml, values.yaml, templates/)
├── Jenkinsfile              # CI/CD pipeline
├── docs/
│   ├── ARCHITECTURE.md      # Diagrams
│   └── screenshots/         # Evidence for each validation step
└── README.md
```

## Prerequisites

AWS CLI v2 (configured), Docker, kubectl, Helm 3, eksctl, and an AWS account with rights for ECR, EKS, EC2, IAM and CloudWatch.

```bash
aws configure
aws sts get-caller-identity      # confirm the account
export AWS_REGION=us-east-1
export ACCOUNT_ID=<your-account-id>
export ECR=$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
```

## Deployment, step by step

### 1. Create ECR repositories and push images

```bash
for r in streaming-auth streaming-streaming streaming-admin streaming-chat streaming-frontend; do
  aws ecr create-repository --repository-name $r --region $AWS_REGION
done

aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR

TAG=1.0.0
docker build -t $ECR/streaming-auth:$TAG      backend/authService
docker build -t $ECR/streaming-streaming:$TAG -f backend/streamingService/Dockerfile backend
docker build -t $ECR/streaming-admin:$TAG     -f backend/adminService/Dockerfile backend
docker build -t $ECR/streaming-chat:$TAG      -f backend/chatService/Dockerfile backend
docker build -t $ECR/streaming-frontend:$TAG  \
  --build-arg REACT_APP_AUTH_API_URL=http://<INGRESS_HOST>/api/auth \
  --build-arg REACT_APP_STREAMING_API_URL=http://<INGRESS_HOST>/api \
  frontend
for r in auth streaming admin chat frontend; do docker push $ECR/streaming-$r:$TAG; done
```

> The frontend inlines API URLs at build time. Build it with the same host you will use for the Ingress. Jenkins does this automatically (see below).

### 2. Create the EKS cluster

```bash
eksctl create cluster --name streamingapp-eks --region us-east-1 \
  --nodegroup-name spot-nodes --node-type t3.medium --nodes 2 --spot
aws eks update-kubeconfig --region us-east-1 --name streamingapp-eks
kubectl get nodes
```

### 3. Install the Ingress controller (EKS does not ship one)

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --set controller.service.type=LoadBalancer

kubectl get svc -n ingress-nginx ingress-nginx-controller   # note EXTERNAL-IP (ELB hostname)
```

### 4. Deploy the Helm chart

```bash
helm install streamingapp ./streamingapp \
  --set ingress.host=<INGRESS_HOST> \
  --set services.auth.image.tag=$TAG \
  --set services.streaming.image.tag=$TAG \
  --set services.admin.image.tag=$TAG \
  --set services.chat.image.tag=$TAG \
  --set services.frontend.image.tag=$TAG

kubectl get pods,svc,ingress -A
```

Later releases use `helm upgrade streamingapp ./streamingapp --set services.<name>.image.tag=<new-tag>`.

### 5. Reach the app through Ingress

`<INGRESS_HOST>` must resolve to the ELB.

- **Option A (hosts file):** resolve the ELB hostname to an IP (`nslookup <elb-hostname>`) and add `<ip> streamingapp.eks` to your hosts file.
- **Option B (no DNS setup):** use the ELB hostname itself as `INGRESS_HOST`. Build the frontend and install the chart with that value.

Then open `http://<INGRESS_HOST>/`.

## CI/CD with Jenkins

The [`Jenkinsfile`](Jenkinsfile) does the following:

1. Checks out the repo and computes `IMAGE_TAG = 1.0.<build number>-<7-char git SHA>`.
2. Logs in to ECR.
3. Builds and pushes all 5 images (each tagged with `IMAGE_TAG` and `latest`). The frontend gets `--build-arg` API URLs derived from the `INGRESS_HOST` parameter.
4. If `DEPLOY_TO_EKS` is ticked: runs `aws eks update-kubeconfig` and `helm upgrade --install --wait` with the new tags for every service.
5. Optionally publishes a success or failure message to an SNS topic (bonus ChatOps).

**Jenkins setup**

1. *Manage Jenkins → Credentials → Global*: add two **Secret text** credentials, `HARSHRAJ_AWS_ACCESS_KEY_ID` and `HARSHRAJ_AWS_SECRET_ACCESS_KEY` (rename in the Jenkinsfile if you use different IDs).
2. *New Item → Pipeline* → **Pipeline script from SCM** → Git → your fork URL → branch `*/main` → script path `Jenkinsfile`.
3. Run once manually so Jenkins reads the `triggers` and `parameters` blocks.

**Automatic trigger on commit**

- In GitHub: *Settings → Webhooks → Add webhook*
  - Payload URL: `https://jenkinsacademics.herovired.com/github-webhook/`
  - Content type: `application/json`
  - Event: *Just the push event*
- If the webhook cannot reach Jenkins, the pipeline still polls SCM every ~5 minutes (`pollSCM('H/5 * * * *')`).

## Monitoring and logging

- **Logs:** application logs are centralized in CloudWatch Logs (log groups per service).
- **Metrics and alarms:** CloudWatch alarm `EKS-High-CPU-Utilization` (CPUUtilization > 80% for 5 minutes) on the node group's Auto Scaling Group. Find it with:

```bash
aws autoscaling describe-auto-scaling-groups \
  --query "AutoScalingGroups[?contains(AutoScalingGroupName,'streamingapp-eks')].AutoScalingGroupName" --output text
```

- **ChatOps (bonus):** create an SNS topic, subscribe it to Slack/Teams/Telegram (via AWS Chatbot or a Lambda webhook), and pass its ARN as the `SNS_TOPIC_ARN` Jenkins parameter.

## Scaling, rolling updates and self-healing

```bash
kubectl scale deploy/streaming --replicas=4
kubectl rollout status deploy/streaming

helm upgrade streamingapp ./streamingapp --set services.auth.image.tag=<new-tag> --wait
kubectl rollout status deploy/auth

kubectl delete pod <auth-pod-name>      # Deployment recreates it
kubectl get pods -w
```

Every Deployment uses `RollingUpdate` with `maxUnavailable: 0` and `maxSurge: 1`, plus readiness/liveness probes, so old pods stay in service until new ones are Ready.

## Validation checklist

| Check | Evidence |
|---|---|
| All pods Running/Ready, Ingress has an ADDRESS | `docs/screenshots/01-get-pods-svc-ingress.png` |
| Register and log in (JWT received) | `docs/screenshots/02-login.png` |
| Admin upload | `docs/screenshots/03-upload.png` |
| Video playback from catalogue | `docs/screenshots/04-playback.png` |
| Chat across two tabs | `docs/screenshots/05-chat-two-tabs.png` |
| Scale to 4 replicas | `docs/screenshots/06-scale.png` |
| Rolling update, zero downtime | `docs/screenshots/07-rolling-update.png` |
| Pod self-heals | `docs/screenshots/08-self-heal.png` |
| Green Jenkins build and webhook trigger | `docs/screenshots/09-jenkins.png` |
| CloudWatch logs and alarm | `docs/screenshots/10-cloudwatch.png` |

## Known limitations

- **MongoDB uses `emptyDir`.** The EBS CSI driver could not be made healthy on this cluster, so data is lost if the Mongo pod restarts. Acceptable for a demo; see production notes.
- **S3 uploads:** the app needs a real `AWS_S3_BUCKET` and credentials for admin uploads. Where no bucket is configured, sample videos were seeded directly into MongoDB.
- **HTTP only:** no TLS is configured.
- **Spot nodes** can be reclaimed at any time.

## Production considerations

For a production cluster I would deploy each environment into its own namespace, with resource requests/limits and NetworkPolicies isolating the services from one another. TLS would be terminated at the Ingress with cert-manager and Let's Encrypt behind a Route 53 domain, and the frontend would be built with relative API paths so the image is not tied to one hostname. Horizontal Pod Autoscalers (with metrics-server) and the Cluster Autoscaler would replace manual scaling, alongside PodDisruptionBudgets on on-demand nodes rather than spot. MongoDB would move to a managed service (DocumentDB or Atlas), or to a replicated StatefulSet on EBS gp3 volumes with backups. Secrets would come from AWS Secrets Manager via External Secrets and workloads would use IRSA roles instead of long-lived IAM keys, both in the pods and in Jenkins. Finally, images would be scanned in ECR and promoted through the pipeline with immutable tags and an approval gate before production.

## Cleanup (avoid AWS charges)

```bash
helm uninstall streamingapp
helm uninstall ingress-nginx -n ingress-nginx   # deletes the ELB
eksctl delete cluster --name streamingapp-eks --region us-east-1
```
