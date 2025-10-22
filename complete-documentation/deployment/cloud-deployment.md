# Cloud Deployment

Complete guide for deploying RamAPI applications to major cloud platforms.

> **Note**: This guide contains documentation examples showing how to deploy RamAPI applications to various cloud platforms. The infrastructure configurations (Docker, Kubernetes, cloud platform setups) are production-ready and follow industry best practices. RamAPI-specific configuration has been verified against the source code.

## Table of Contents

1. [AWS Deployment](#aws-deployment)
2. [Google Cloud Deployment](#google-cloud-deployment)
3. [Azure Deployment](#azure-deployment)
4. [Vercel Deployment](#vercel-deployment)
5. [Railway Deployment](#railway-deployment)
6. [Fly.io Deployment](#fly-io-deployment)
7. [Platform Comparison](#platform-comparison)

---

## AWS Deployment

### AWS ECS (Elastic Container Service)

**Best for**: Production-grade containerized applications with full control

#### Prerequisites

```bash
# Install AWS CLI
brew install awscli  # macOS
# or
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"

# Configure AWS
aws configure
```

#### 1. Create ECR Repository

```bash
# Create repository
aws ecr create-repository --repository-name ramapi-app

# Login to ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  <account-id>.dkr.ecr.us-east-1.amazonaws.com

# Build and push image
docker build -t ramapi-app .
docker tag ramapi-app:latest <account-id>.dkr.ecr.us-east-1.amazonaws.com/ramapi-app:latest
docker push <account-id>.dkr.ecr.us-east-1.amazonaws.com/ramapi-app:latest
```

#### 2. Task Definition

```json
{
  "family": "ramapi-app",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "containerDefinitions": [
    {
      "name": "ramapi",
      "image": "<account-id>.dkr.ecr.us-east-1.amazonaws.com/ramapi-app:latest",
      "portMappings": [
        {
          "containerPort": 3000,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {
          "name": "NODE_ENV",
          "value": "production"
        },
        {
          "name": "PORT",
          "value": "3000"
        }
      ],
      "secrets": [
        {
          "name": "JWT_SECRET",
          "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789012:secret:jwt-secret"
        },
        {
          "name": "DATABASE_URL",
          "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789012:secret:database-url"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/ramapi-app",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      },
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:3000/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 60
      }
    }
  ]
}
```

#### 3. Create ECS Service

```bash
# Create cluster
aws ecs create-cluster --cluster-name ramapi-cluster

# Register task definition
aws ecs register-task-definition --cli-input-json file://task-definition.json

# Create service with load balancer
aws ecs create-service \
  --cluster ramapi-cluster \
  --service-name ramapi-service \
  --task-definition ramapi-app \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-12345,subnet-67890],securityGroups=[sg-12345],assignPublicIp=ENABLED}" \
  --load-balancers "targetGroupArn=arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/ramapi-tg/1234567890,containerName=ramapi,containerPort=3000"
```

#### 4. Auto Scaling

```bash
# Register scalable target
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --resource-id service/ramapi-cluster/ramapi-service \
  --scalable-dimension ecs:service:DesiredCount \
  --min-capacity 2 \
  --max-capacity 10

# Create scaling policy
aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --resource-id service/ramapi-cluster/ramapi-service \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-name cpu-scaling \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration file://scaling-policy.json
```

**scaling-policy.json:**

```json
{
  "TargetValue": 70.0,
  "PredefinedMetricSpecification": {
    "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
  },
  "ScaleInCooldown": 300,
  "ScaleOutCooldown": 60
}
```

---

### AWS Lambda (Serverless)

**Best for**: Event-driven APIs, low-traffic applications

> **Note**: The Lambda adapter example below is conceptual and may require additional implementation. RamAPI does not currently expose a `handleRequest()` method. For serverless deployment, you may need to adapt the server to work with Lambda's request/response format or use a serverless framework adapter.

#### Lambda Handler

```typescript
// lambda.ts
import { Server } from 'ramapi';
import { APIGatewayProxyEvent, APIGatewayProxyResult, Context } from 'aws-lambda';

let server: Server;

async function getServer() {
  if (!server) {
    const { createApp } = await import('./app.js');
    server = createApp({ adapter: { type: 'node-http' } });
  }
  return server;
}

export const handler = async (
  event: APIGatewayProxyEvent,
  context: Context
): Promise<APIGatewayProxyResult> => {
  const app = await getServer();

  // Convert API Gateway event to standard request
  const request = {
    method: event.httpMethod,
    url: event.path + (event.queryStringParameters ? '?' + new URLSearchParams(event.queryStringParameters).toString() : ''),
    headers: event.headers,
    body: event.body ? (event.isBase64Encoded ? Buffer.from(event.body, 'base64') : event.body) : undefined,
  };

  // Process request
  const response = await app.handleRequest(request);

  return {
    statusCode: response.status,
    headers: response.headers,
    body: response.body,
    isBase64Encoded: false,
  };
};
```

#### Deploy with SAM

```yaml
# template.yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Globals:
  Function:
    Timeout: 30
    MemorySize: 512
    Runtime: nodejs20.x
    Environment:
      Variables:
        NODE_ENV: production

Resources:
  RamAPIFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: dist/
      Handler: lambda.handler
      Events:
        ApiEvent:
          Type: HttpApi
          Properties:
            Path: /{proxy+}
            Method: ANY
      Environment:
        Variables:
          JWT_SECRET: !Ref JWTSecret
          DATABASE_URL: !Ref DatabaseURL

  JWTSecret:
    Type: AWS::SecretsManager::Secret
    Properties:
      Name: ramapi-jwt-secret
      SecretString: !Sub '{"secret":"${AWS::StackName}-jwt-secret"}'

Outputs:
  ApiUrl:
    Description: "API Gateway endpoint URL"
    Value: !Sub "https://${ServerlessHttpApi}.execute-api.${AWS::Region}.amazonaws.com/"
```

**Deploy:**

```bash
# Build
npm run build

# Deploy
sam build
sam deploy --guided
```

---

### AWS Elastic Beanstalk

**Best for**: Simple deployment without managing infrastructure

#### 1. Create Application

```bash
# Initialize
eb init -p node.js-20 ramapi-app --region us-east-1

# Create environment
eb create ramapi-prod --instance-type t3.small --single
```

#### 2. Configuration

**.ebextensions/01-environment.config:**

```yaml
option_settings:
  aws:elasticbeanstalk:application:environment:
    NODE_ENV: production
    PORT: 8080
  aws:elasticbeanstalk:container:nodejs:
    NodeCommand: "node dist/index.js"
  aws:autoscaling:asg:
    MinSize: 2
    MaxSize: 10
  aws:autoscaling:trigger:
    MeasureName: CPUUtilization
    Statistic: Average
    Unit: Percent
    UpperThreshold: 70
    LowerThreshold: 30
```

#### 3. Deploy

```bash
# Deploy
eb deploy

# View logs
eb logs

# Open in browser
eb open
```

---

## Google Cloud Deployment

### Cloud Run (Serverless Containers)

**Best for**: Containerized apps with auto-scaling, pay-per-use

#### 1. Create Dockerfile

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY dist ./dist
ENV PORT=8080
CMD ["node", "dist/index.js"]
```

#### 2. Deploy to Cloud Run

```bash
# Set project
gcloud config set project YOUR_PROJECT_ID

# Build and deploy
gcloud run deploy ramapi-app \
  --source . \
  --platform managed \
  --region us-central1 \
  --allow-unauthenticated \
  --set-env-vars NODE_ENV=production \
  --set-secrets JWT_SECRET=jwt-secret:latest,DATABASE_URL=database-url:latest \
  --min-instances 1 \
  --max-instances 10 \
  --cpu 1 \
  --memory 512Mi \
  --concurrency 80 \
  --timeout 300
```

#### 3. Custom Domain

```bash
# Map custom domain
gcloud run domain-mappings create \
  --service ramapi-app \
  --domain api.yourdomain.com \
  --region us-central1
```

#### 4. Cloud Run YAML

```yaml
# cloudrun.yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: ramapi-app
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/minScale: '1'
        autoscaling.knative.dev/maxScale: '10'
    spec:
      containerConcurrency: 80
      timeoutSeconds: 300
      containers:
      - image: gcr.io/YOUR_PROJECT/ramapi-app
        ports:
        - containerPort: 8080
        env:
        - name: NODE_ENV
          value: production
        - name: PORT
          value: "8080"
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: jwt-secret
              key: latest
        resources:
          limits:
            cpu: "1"
            memory: 512Mi
```

**Deploy:**

```bash
gcloud run services replace cloudrun.yaml --region us-central1
```

---

### Google Kubernetes Engine (GKE)

**Best for**: Complex microservices, full Kubernetes control

#### 1. Create GKE Cluster

```bash
# Create cluster
gcloud container clusters create ramapi-cluster \
  --num-nodes=3 \
  --machine-type=n1-standard-2 \
  --region=us-central1 \
  --enable-autoscaling \
  --min-nodes=1 \
  --max-nodes=10

# Get credentials
gcloud container clusters get-credentials ramapi-cluster --region us-central1
```

#### 2. Kubernetes Manifests

**deployment.yaml:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ramapi-app
  labels:
    app: ramapi
spec:
  replicas: 3
  selector:
    matchLabels:
      app: ramapi
  template:
    metadata:
      labels:
        app: ramapi
    spec:
      containers:
      - name: ramapi
        image: gcr.io/YOUR_PROJECT/ramapi-app:latest
        ports:
        - containerPort: 3000
        env:
        - name: NODE_ENV
          value: production
        - name: PORT
          value: "3000"
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: ramapi-secrets
              key: jwt-secret
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: ramapi-secrets
              key: database-url
        resources:
          requests:
            cpu: 250m
            memory: 512Mi
          limits:
            cpu: 500m
            memory: 1Gi
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: ramapi-service
spec:
  type: LoadBalancer
  selector:
    app: ramapi
  ports:
  - port: 80
    targetPort: 3000
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: ramapi-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ramapi-app
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

#### 3. Deploy

```bash
# Create secrets
kubectl create secret generic ramapi-secrets \
  --from-literal=jwt-secret=YOUR_JWT_SECRET \
  --from-literal=database-url=YOUR_DATABASE_URL

# Deploy
kubectl apply -f deployment.yaml

# Get external IP
kubectl get service ramapi-service
```

---

## Azure Deployment

### Azure App Service

**Best for**: Simple deployment with built-in scaling

#### 1. Create App Service

```bash
# Login
az login

# Create resource group
az group create --name ramapi-rg --location eastus

# Create App Service plan
az appservice plan create \
  --name ramapi-plan \
  --resource-group ramapi-rg \
  --sku P1V2 \
  --is-linux

# Create web app
az webapp create \
  --resource-group ramapi-rg \
  --plan ramapi-plan \
  --name ramapi-app \
  --runtime "NODE:20-lts"
```

#### 2. Configure Application

```bash
# Set environment variables
az webapp config appsettings set \
  --resource-group ramapi-rg \
  --name ramapi-app \
  --settings \
    NODE_ENV=production \
    PORT=8080 \
    JWT_SECRET=@Microsoft.KeyVault(SecretUri=https://your-vault.vault.azure.net/secrets/jwt-secret/) \
    DATABASE_URL=@Microsoft.KeyVault(SecretUri=https://your-vault.vault.azure.net/secrets/database-url/)

# Configure startup command
az webapp config set \
  --resource-group ramapi-rg \
  --name ramapi-app \
  --startup-file "node dist/index.js"
```

#### 3. Deploy

```bash
# Deploy from local Git
az webapp deployment source config-local-git \
  --name ramapi-app \
  --resource-group ramapi-rg

# Add Azure remote
git remote add azure <deployment-url>

# Push to deploy
git push azure main
```

#### 4. Auto Scaling

```bash
# Create autoscale rule
az monitor autoscale create \
  --resource-group ramapi-rg \
  --resource ramapi-plan \
  --resource-type Microsoft.Web/serverfarms \
  --name ramapi-autoscale \
  --min-count 2 \
  --max-count 10 \
  --count 2

# Scale out rule (CPU > 70%)
az monitor autoscale rule create \
  --resource-group ramapi-rg \
  --autoscale-name ramapi-autoscale \
  --condition "Percentage CPU > 70 avg 5m" \
  --scale out 1

# Scale in rule (CPU < 30%)
az monitor autoscale rule create \
  --resource-group ramapi-rg \
  --autoscale-name ramapi-autoscale \
  --condition "Percentage CPU < 30 avg 5m" \
  --scale in 1
```

---

### Azure Container Instances

**Best for**: Quick container deployment without orchestration

#### 1. Create Container Instance

```bash
# Create container
az container create \
  --resource-group ramapi-rg \
  --name ramapi-container \
  --image YOUR_REGISTRY/ramapi-app:latest \
  --dns-name-label ramapi-app \
  --ports 3000 \
  --cpu 2 \
  --memory 4 \
  --environment-variables \
    NODE_ENV=production \
    PORT=3000 \
  --secure-environment-variables \
    JWT_SECRET=YOUR_JWT_SECRET \
    DATABASE_URL=YOUR_DATABASE_URL
```

#### 2. YAML Deployment

```yaml
# container-instance.yaml
apiVersion: 2021-09-01
location: eastus
name: ramapi-container
properties:
  containers:
  - name: ramapi
    properties:
      image: YOUR_REGISTRY/ramapi-app:latest
      resources:
        requests:
          cpu: 2
          memoryInGb: 4
      ports:
      - port: 3000
        protocol: TCP
      environmentVariables:
      - name: NODE_ENV
        value: production
      - name: PORT
        value: "3000"
      - name: JWT_SECRET
        secureValue: YOUR_JWT_SECRET
      - name: DATABASE_URL
        secureValue: YOUR_DATABASE_URL
  osType: Linux
  ipAddress:
    type: Public
    dnsNameLabel: ramapi-app
    ports:
    - protocol: TCP
      port: 3000
tags: {}
type: Microsoft.ContainerInstance/containerGroups
```

**Deploy:**

```bash
az container create --resource-group ramapi-rg --file container-instance.yaml
```

---

### Azure Kubernetes Service (AKS)

**Best for**: Production-grade Kubernetes on Azure

#### 1. Create AKS Cluster

```bash
# Create cluster
az aks create \
  --resource-group ramapi-rg \
  --name ramapi-aks \
  --node-count 3 \
  --enable-managed-identity \
  --generate-ssh-keys \
  --enable-addons monitoring \
  --enable-cluster-autoscaler \
  --min-count 1 \
  --max-count 10

# Get credentials
az aks get-credentials --resource-group ramapi-rg --name ramapi-aks
```

#### 2. Deploy (same Kubernetes manifests as GKE)

```bash
kubectl apply -f deployment.yaml
```

---

## Vercel Deployment

**Best for**: Edge functions, serverless APIs, JAMstack

> **Note**: The Vercel adapter examples below are conceptual and may require additional implementation. RamAPI does not currently expose a `handleRequest()` method. Consider using containerized deployment on Vercel or adapting the server to work with Vercel's serverless function format.

### Method 1: Serverless Functions

**api/index.ts:**

```typescript
import { Server } from 'ramapi';
import type { VercelRequest, VercelResponse } from '@vercel/node';

let server: Server;

async function getServer() {
  if (!server) {
    const { createApp } = await import('../src/app.js');
    server = createApp({ adapter: { type: 'node-http' } });
  }
  return server;
}

export default async function handler(req: VercelRequest, res: VercelResponse) {
  const app = await getServer();

  const request = {
    method: req.method || 'GET',
    url: req.url || '/',
    headers: req.headers as Record<string, string>,
    body: req.body,
  };

  const response = await app.handleRequest(request);

  res.status(response.status);
  Object.entries(response.headers).forEach(([key, value]) => {
    res.setHeader(key, value);
  });
  res.send(response.body);
}
```

**vercel.json:**

```json
{
  "version": 2,
  "builds": [
    {
      "src": "api/index.ts",
      "use": "@vercel/node"
    }
  ],
  "routes": [
    {
      "src": "/(.*)",
      "dest": "/api/index.ts"
    }
  ],
  "env": {
    "NODE_ENV": "production"
  }
}
```

**Deploy:**

```bash
# Install Vercel CLI
npm i -g vercel

# Deploy
vercel --prod
```

### Method 2: Edge Functions

**api/[...path].ts:**

```typescript
import type { VercelRequest, VercelResponse } from '@vercel/node';

export const config = {
  runtime: 'edge',
};

export default async function handler(req: VercelRequest) {
  // Your RamAPI logic here (adapted for edge runtime)
  return new Response(JSON.stringify({ message: 'Hello from Edge' }), {
    status: 200,
    headers: { 'Content-Type': 'application/json' },
  });
}
```

---

## Railway Deployment

**Best for**: Simple deployment with GitHub integration

### Method 1: Railway CLI

```bash
# Install Railway CLI
npm i -g @railway/cli

# Login
railway login

# Initialize project
railway init

# Deploy
railway up
```

### Method 2: Railway Config

**railway.json:**

```json
{
  "$schema": "https://railway.app/railway.schema.json",
  "build": {
    "builder": "NIXPACKS",
    "buildCommand": "npm run build"
  },
  "deploy": {
    "startCommand": "node dist/index.js",
    "restartPolicyType": "ON_FAILURE",
    "restartPolicyMaxRetries": 10
  }
}
```

**railway.toml:**

```toml
[build]
builder = "NIXPACKS"
buildCommand = "npm run build"

[deploy]
startCommand = "node dist/index.js"
restartPolicyType = "ON_FAILURE"
restartPolicyMaxRetries = 10
healthcheckPath = "/health"
healthcheckTimeout = 100
```

### GitHub Integration

1. Connect Railway to GitHub repository
2. Railway auto-detects Node.js and deploys
3. Environment variables in Railway dashboard
4. Auto-deploy on push to main

---

## Fly.io Deployment

**Best for**: Global edge deployment, low-latency apps

### 1. Install Fly CLI

```bash
# Install
curl -L https://fly.io/install.sh | sh

# Login
flyctl auth login
```

### 2. Initialize App

```bash
# Initialize
flyctl launch

# This creates fly.toml
```

**fly.toml:**

```toml
app = "ramapi-app"
primary_region = "sjc"

[build]
  builder = "paketobuildpacks/builder:base"
  buildpacks = ["gcr.io/paketo-buildpacks/nodejs"]

[env]
  NODE_ENV = "production"
  PORT = "8080"

[http_service]
  internal_port = 8080
  force_https = true
  auto_stop_machines = true
  auto_start_machines = true
  min_machines_running = 1
  processes = ["app"]

[[services.ports]]
  handlers = ["http"]
  port = 80

[[services.ports]]
  handlers = ["tls", "http"]
  port = 443

[services.concurrency]
  type = "connections"
  hard_limit = 1000
  soft_limit = 800

[[services.http_checks]]
  interval = "10s"
  timeout = "2s"
  grace_period = "5s"
  method = "GET"
  path = "/health"

[services.scaling]
  min_machines = 1
  max_machines = 10

[[vm]]
  cpu_kind = "shared"
  cpus = 1
  memory_mb = 512
```

### 3. Deploy

```bash
# Set secrets
flyctl secrets set JWT_SECRET=your-secret DATABASE_URL=your-db-url

# Deploy
flyctl deploy

# Scale
flyctl scale count 3

# View status
flyctl status

# View logs
flyctl logs
```

### 4. Multi-Region Deployment

```bash
# Add regions
flyctl regions add sea lax ams

# Scale per region
flyctl scale count 2 --region sea
flyctl scale count 2 --region lax
flyctl scale count 1 --region ams
```

---

## Platform Comparison

| Platform | Type | Pricing | Auto-Scale | Cold Start | Best For |
|----------|------|---------|------------|------------|----------|
| **AWS ECS** | Container | Pay for resources | Yes (manual) | No | Production apps |
| **AWS Lambda** | Serverless | Pay per request | Automatic | Yes (~100ms) | Event-driven |
| **AWS Beanstalk** | PaaS | Pay for resources | Yes | No | Simple deployment |
| **Cloud Run** | Serverless Container | Pay per request | Automatic | Yes (~200ms) | Modern containers |
| **GKE** | Kubernetes | Pay for nodes | Yes | No | Complex microservices |
| **Azure App Service** | PaaS | Fixed + scale | Yes | No | Enterprise apps |
| **Azure ACI** | Container | Pay per second | No | No | Simple containers |
| **AKS** | Kubernetes | Pay for nodes | Yes | No | Enterprise K8s |
| **Vercel** | Serverless | Free + pro tiers | Automatic | Yes (~50ms) | Edge/JAMstack |
| **Railway** | PaaS | Usage-based | Automatic | No | Quick deployments |
| **Fly.io** | Edge PaaS | Usage-based | Automatic | No | Global edge apps |

### Cost Comparison (Monthly estimate for ~100K requests)

- **AWS Lambda**: ~$5-20 (very cost-effective for low traffic)
- **Cloud Run**: ~$10-30 (pay-per-use)
- **AWS ECS (Fargate)**: ~$50-150 (2 tasks)
- **Azure App Service**: ~$70-200 (P1V2 plan)
- **GKE/AKS**: ~$150-300 (3-node cluster)
- **Vercel**: Free-$20 (hobby/pro)
- **Railway**: ~$5-50 (usage-based)
- **Fly.io**: ~$5-40 (shared CPUs)

### Performance Comparison

**Latency (p50):**
- Fly.io: ~10-30ms (edge)
- Cloud Run: ~20-50ms
- ECS/GKE/AKS: ~15-40ms
- Vercel Edge: ~10-30ms
- Lambda: ~50-150ms (cold start)

**Throughput:**
- ECS/GKE/AKS: 10K+ req/s (depends on instances)
- Cloud Run: 5K+ req/s (auto-scales)
- Lambda: 1K+ req/s (concurrent limit)
- Fly.io: 5K+ req/s (multi-region)

---

## Best Practices

### 1. Environment Variables

Always use platform secret managers:

```bash
# AWS
aws secretsmanager create-secret --name jwt-secret --secret-string "your-secret"

# Google Cloud
echo -n "your-secret" | gcloud secrets create jwt-secret --data-file=-

# Azure
az keyvault secret set --vault-name your-vault --name jwt-secret --value "your-secret"

# Fly.io
flyctl secrets set JWT_SECRET=your-secret
```

### 2. Health Checks

Implement proper health endpoints:

```typescript
app.get('/health', (ctx) => {
  ctx.json({ status: 'healthy', timestamp: Date.now() });
});

app.get('/ready', async (ctx) => {
  // Check database connection
  const dbOk = await checkDatabase();
  if (!dbOk) {
    ctx.json({ status: 'not ready', reason: 'database' }, 503);
    return;
  }
  ctx.json({ status: 'ready' });
});
```

### 3. Logging

Use structured logging for cloud platforms:

```typescript
console.log(JSON.stringify({
  level: 'info',
  message: 'Request processed',
  traceId: ctx.trace?.traceId,
  duration: 123,
  path: ctx.path,
  method: ctx.method,
  status: 200,
}));
```

### 4. Graceful Shutdown

Handle termination signals:

```typescript
const shutdown = async (signal: string) => {
  console.log(`${signal} received, shutting down gracefully`);
  await app.close();
  process.exit(0);
};

process.on('SIGTERM', () => shutdown('SIGTERM'));
process.on('SIGINT', () => shutdown('SIGINT'));
```

### 5. Database Connection Pooling

Optimize for serverless:

```typescript
// For serverless (Lambda, Cloud Run)
const pool = new Pool({
  max: 1, // Single connection per instance
  idleTimeoutMillis: 300000, // Keep alive
});

// For containers (ECS, GKE)
const pool = new Pool({
  max: 10, // More connections
  idleTimeoutMillis: 30000,
});
```

---

## See Also

- [Docker Deployment](docker.md)
- [Production Setup](production-setup.md)
- [Production Observability](production-observability.md)
