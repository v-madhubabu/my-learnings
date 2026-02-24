# my-project

STEP 1 — Prerequisites

Install:
Docker
kubectl
gcloud CLI
Helm
Node.js
Git

commands:
gcloud auth login
gcloud config set project YOUR_PROJECT_ID

enable apis:
<img width="1909" height="933" alt="image" src="https://github.com/user-attachments/assets/5f28c659-bc4f-4faa-967e-247bac7051b9" />


STEP 2
Create GKE Cluster
gcloud container clusters create devops-cluster \
  --num-nodes=3 \
  --machine-type=e2-medium \
  --zone=us-central1-a
connect:
  gcloud container clusters get-credentials devops-cluster --zone us-central1-a
 check  kubectl get nodes

step 3
 Create Microservices
We’ll create 3 services:
frontend
user-service
order-service

  user-service/index.js
const express = require('express');
const app = express();
app.use(express.json());

app.get('/users', (req, res) => {
    res.json([{ id: 1, name: "John Doe" }]);
});

app.listen(3000, () => console.log("User service running on port 3000"));

package.json
{
  "name": "user-service",
  "version": "1.0.0",
  "main": "index.js",
  "dependencies": {
    "express": "^4.18.2"
  }
}

order-service
const express = require('express');
const app = express();

app.get('/orders', (req, res) => {
    res.json([{ id: 101, item: "Laptop" }]);
});

app.listen(3000, () => console.log("Order service running"));

build docker image

FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["node", "index.js"]

Build and push to GCR:

gcloud auth configure-docker
docker build -t gcr.io/YOUR_PROJECT_ID/user-service:v1 .
docker push gcr.io/YOUR_PROJECT_ID/user-service:v1

apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
spec:
  replicas: 2
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
    spec:
      containers:
      - name: user-service
        image: gcr.io/YOUR_PROJECT_ID/user-service:v1
        ports:
        - containerPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: user-service
spec:
  selector:
    app: user-service
  ports:
  - port: 80
    targetPort: 3000
kubectl apply -f user-deployment.yaml

STEP 6 — Install Istio
Download Istio:
curl -L https://istio.io/downloadIstio | sh -
cd istio-*
istioctl install --set profile=demo -y
kubectl label namespace default istio-injection=enabled

STEP 7 — Create Istio Gateway
gateway.yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: app-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
Virtual Service
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: app-routing
spec:
  hosts:
  - "*"
  gateways:
  - app-gateway
  http:
  - match:
    - uri:
        prefix: /users
    route:
    - destination:
        host: user-service
        port:
          number: 80
  - match:
    - uri:
        prefix: /orders
    route:
    - destination:
        host: order-service
        port:
          number: 80

Apply:

kubectl apply -f gateway.yaml
kubectl apply -f virtualservice.yaml
🚀 STEP 8 — Get External IP
kubectl get svc istio-ingressgateway -n istio-system

Open:

http://EXTERNAL_IP/users
http://EXTERNAL_IP/orders
🚀 STEP 9 — Add DevOps CI/CD (GitHub Actions)
.github/workflows/deploy.yml
name: Deploy to GKE

on:
  push:
    branches:
      - main

jobs:
  build-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: google-github-actions/setup-gcloud@v1
        with:
          project_id: ${{ secrets.GCP_PROJECT }}
          service_account_key: ${{ secrets.GCP_SA_KEY }}

      - run: |
          docker build -t gcr.io/$GCP_PROJECT/user-service:$GITHUB_SHA ./user-service
          docker push gcr.io/$GCP_PROJECT/user-service:$GITHUB_SHA

      - run: |
          kubectl set image deployment/user-service \
          user-service=gcr.io/$GCP_PROJECT/user-service:$GITHUB_SHA
    
