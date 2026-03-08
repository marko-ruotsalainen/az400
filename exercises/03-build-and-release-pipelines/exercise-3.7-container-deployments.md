# Exercise 3.7: Container Deployments

## Objective
Build, push, and deploy containerized applications using Docker, ACR, and AKS.

## Skills Measured
- Implement application deployment by using containers, binaries, and scripts
- Automate container scanning

## Steps

### Part A: Create a Dockerfile

Create `Dockerfile`:
```dockerfile
# Multi-stage build for smaller image
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .

FROM node:20-alpine
WORKDIR /app
# Run as non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
COPY --from=build /app .
USER appuser
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s CMD wget -qO- http://localhost:3000/health || exit 1
CMD ["node", "hello.js"]
```

Create `.dockerignore`:
```
node_modules
npm-debug.log
.git
.github
*.md
coverage
test
```

### Part B: Build and Push to Azure Container Registry

```bash
# Create ACR
az acr create --name az400acr --resource-group rg-az400 --sku Basic

# Build locally
docker build -t az400acr.azurecr.io/hello-app:1.0.0 .
docker build -t az400acr.azurecr.io/hello-app:latest .

# Login and push
az acr login --name az400acr
docker push az400acr.azurecr.io/hello-app:1.0.0
docker push az400acr.azurecr.io/hello-app:latest

# Or build directly in ACR (no local Docker needed!)
az acr build --registry az400acr --image hello-app:1.0.0 .
```

### Part C: Container Build in Azure Pipelines

```yaml
# cicd/docker-pipeline.yaml
trigger:
  branches:
    include: [main]

variables:
  acrName: 'az400acr'
  imageName: 'hello-app'
  tag: '$(Build.BuildId)'

stages:
  - stage: Build
    jobs:
      - job: BuildImage
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - task: Docker@2
            displayName: 'Build and Push'
            inputs:
              containerRegistry: 'ACR-Connection'
              repository: $(imageName)
              command: 'buildAndPush'
              Dockerfile: '**/Dockerfile'
              tags: |
                $(tag)
                latest

  - stage: Deploy
    dependsOn: Build
    jobs:
      - deployment: DeployToAKS
        environment: 'Production'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: KubernetesManifest@1
                  inputs:
                    action: 'deploy'
                    namespace: 'default'
                    manifests: 'k8s/deployment.yaml'
                    containers: '$(acrName).azurecr.io/$(imageName):$(tag)'
```

### Part D: Container Build in GitHub Actions

```yaml
name: Docker Build and Deploy
on:
  push:
    branches: [main]

env:
  ACR_NAME: az400acr
  IMAGE_NAME: hello-app

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: azure/login@v2
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - uses: azure/docker-login@v2
        with:
          login-server: ${{ env.ACR_NAME }}.azurecr.io
          username: ${{ secrets.ACR_USERNAME }}
          password: ${{ secrets.ACR_PASSWORD }}

      - run: |
          docker build -t ${{ env.ACR_NAME }}.azurecr.io/${{ env.IMAGE_NAME }}:${{ github.run_number }} .
          docker push ${{ env.ACR_NAME }}.azurecr.io/${{ env.IMAGE_NAME }}:${{ github.run_number }}

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: azure/login@v2
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - uses: azure/aks-set-context@v4
        with:
          resource-group: rg-az400
          cluster-name: aks-az400

      - uses: azure/k8s-deploy@v5
        with:
          manifests: k8s/deployment.yaml
          images: ${{ env.ACR_NAME }}.azurecr.io/${{ env.IMAGE_NAME }}:${{ github.run_number }}
```

### Part E: Kubernetes Manifests

Create `k8s/deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello-app
  template:
    metadata:
      labels:
        app: hello-app
    spec:
      containers:
        - name: hello-app
          image: az400acr.azurecr.io/hello-app:latest
          ports:
            - containerPort: 3000
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 250m
              memory: 256Mi
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 30
          readinessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: hello-app-service
spec:
  type: LoadBalancer
  selector:
    app: hello-app
  ports:
    - port: 80
      targetPort: 3000
```

### Part F: Deploy to Azure Container Apps (Serverless Containers)

```bash
# Create Container Apps environment
az containerapp env create \
  --name az400-env \
  --resource-group rg-az400 \
  --location eastus

# Deploy container
az containerapp create \
  --name hello-app \
  --resource-group rg-az400 \
  --environment az400-env \
  --image az400acr.azurecr.io/hello-app:latest \
  --target-port 3000 \
  --ingress external \
  --registry-server az400acr.azurecr.io
```

## Validation Checklist

- [ ] Multi-stage Dockerfile created
- [ ] Image built and pushed to ACR
- [ ] ACR build (remote build) tested
- [ ] Azure Pipelines Docker task configured
- [ ] GitHub Actions container build workflow created
- [ ] Kubernetes deployment manifest with health probes
- [ ] Container deployment to AKS
- [ ] Container Apps deployment understood

## Microsoft Learn References
- [Build and push Docker images](https://learn.microsoft.com/en-us/azure/devops/pipelines/ecosystems/containers/build-image)
- [Deploy to AKS](https://learn.microsoft.com/en-us/azure/devops/pipelines/ecosystems/kubernetes/aks-template)
