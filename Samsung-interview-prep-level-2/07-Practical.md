Absolutely — let’s turn this into a **real Samsung-style lab** you can do in your GCP trial account over the next 2 days.

The goal is to practice the full path: **GitHub repo → GitHub Actions builds image → pushes image → deploys to GKE → exposes app → you break it → you fix it.** GitHub Actions is built for CI/CD workflows on pushes and merges, and GKE supports deploying apps and exposing them with a `LoadBalancer` service or Ingress. For auth from GitHub Actions to Google Cloud, the modern approach is **Workload Identity Federation** instead of storing long-lived service account keys. ([GitHub Docs](https://docs.github.com/en/actions/get-started/quickstart?utm_source=chatgpt.com))

## **What you will build**

A tiny **Hello World Node.js app** running in Docker. The pipeline will:

1. build the Docker image  
2. push it to **Artifact Registry**  
3. update Kubernetes by running `kubectl apply` against your GKE cluster

For interview practice, this is better than overcomplicating it with Helm or ArgoCD on day one. Once this works, the same flow can be extended into Helm and GitOps. GitHub’s official docs show the standard Docker build/push pattern using `docker/build-push-action`, and GKE’s quickstart shows the basic deploy-and-expose workflow. ([GitHub Docs](https://docs.github.com/actions/guides/publishing-docker-images?utm_source=chatgpt.com))

---

# **Practice architecture**

Developer push to GitHub  
        │  
        ▼  
GitHub Actions  
  ├─ checkout code  
  ├─ authenticate to Google Cloud  
  ├─ build Docker image  
  ├─ push to Artifact Registry  
  ├─ get GKE credentials  
  └─ kubectl apply deployment \+ service  
        │  
        ▼  
GKE Cluster  
  ├─ Deployment  
  ├─ Pods  
  └─ Service type LoadBalancer  
        │  
        ▼  
External IP

This is a valid and common practice flow: GitHub Actions handles build/deploy automation, Artifact Registry stores images, and GKE exposes the application through Kubernetes objects. ([GitHub Docs](https://docs.github.com/en/actions/get-started/quickstart?utm_source=chatgpt.com))

---

# **Repo structure**

Create a new repo like this:

hello-gke-github-actions/  
├── app/  
│   ├── package.json  
│   ├── server.js  
│   └── Dockerfile  
├── k8s/  
│   ├── deployment.yaml  
│   └── service.yaml  
└── .github/  
    └── workflows/  
        └── deploy.yml

---

# **1\) Create the Hello World app**

## **`app/package.json`**

{  
  "name": "hello-gke",  
  "version": "1.0.0",  
  "description": "Simple hello world app for GKE practice",  
  "main": "server.js",  
  "scripts": {  
    "start": "node server.js"  
  },  
  "dependencies": {  
    "express": "^4.19.2"  
  }  
}

## **`app/server.js`**

const express \= require("express");  
const app \= express();

const port \= process.env.PORT || 8080;

app.get("/", (req, res) \=\> {  
  res.send("Hello from GKE via GitHub Actions\!");  
});

app.get("/health", (req, res) \=\> {  
  res.status(200).send("ok");  
});

app.listen(port, () \=\> {  
  console.log(\`App listening on port ${port}\`);  
});

## **`app/Dockerfile`**

FROM node:20-alpine

WORKDIR /app

COPY package.json ./  
RUN npm install \--omit=dev

COPY server.js ./

EXPOSE 8080

CMD \["npm", "start"\]

---

# **2\) Create Kubernetes manifests**

## **`k8s/deployment.yaml`**

Replace `IMAGE_PLACEHOLDER` later from the workflow.

apiVersion: apps/v1  
kind: Deployment  
metadata:  
  name: hello-gke  
spec:  
  replicas: 2  
  selector:  
    matchLabels:  
      app: hello-gke  
  template:  
    metadata:  
      labels:  
        app: hello-gke  
    spec:  
      containers:  
        \- name: hello-gke  
          image: IMAGE\_PLACEHOLDER  
          ports:  
            \- containerPort: 8080  
          readinessProbe:  
            httpGet:  
              path: /health  
              port: 8080  
            initialDelaySeconds: 5  
            periodSeconds: 5  
          livenessProbe:  
            httpGet:  
              path: /health  
              port: 8080  
            initialDelaySeconds: 10  
            periodSeconds: 10

## **`k8s/service.yaml`**

apiVersion: v1  
kind: Service  
metadata:  
  name: hello-gke-service  
spec:  
  type: LoadBalancer  
  selector:  
    app: hello-gke  
  ports:  
    \- port: 80  
      targetPort: 8080

A `LoadBalancer` service on GKE provisions a Google Cloud load balancer for external access. Readiness and liveness probes are standard Kubernetes health checks that control whether traffic should go to a pod and whether a pod should be restarted. ([Google Cloud Documentation](https://docs.cloud.google.com/kubernetes-engine/docs/concepts/service-load-balancer?utm_source=chatgpt.com))

---

# **3\) Prepare GCP once**

Use your trial project.

## **Set variables locally**

export PROJECT\_ID="your-gcp-project-id"  
export REGION="us-central1"  
export ZONE="us-central1-a"  
export CLUSTER\_NAME="hello-gke-cluster"  
export REPO\_NAME="hello-gke-repo"

## **Enable APIs**

gcloud services enable \\  
  container.googleapis.com \\  
  artifactregistry.googleapis.com \\  
  cloudresourcemanager.googleapis.com \\  
  iamcredentials.googleapis.com \\  
  sts.googleapis.com

## **Create Artifact Registry repo**

gcloud artifacts repositories create $REPO\_NAME \\  
  \--repository-format=docker \\  
  \--location=$REGION \\  
  \--description="Docker repo for hello-gke practice"

## **Create a GKE cluster**

gcloud container clusters create $CLUSTER\_NAME \\  
  \--zone $ZONE \\  
  \--num-nodes 2

Then connect:

gcloud container clusters get-credentials $CLUSTER\_NAME \--zone $ZONE

Google’s GKE quickstart documents creating a cluster, deploying a workload, and exposing it. Artifact Registry is the recommended image registry for GCP container images. ([Google Cloud Documentation](https://docs.cloud.google.com/kubernetes-engine/docs/deploy-app-cluster?utm_source=chatgpt.com))

---

# **4\) Set up GitHub Actions auth to GCP**

For interview practice, do this the better way: **Workload Identity Federation**. Google documents this pattern for GitHub Actions so you avoid long-lived JSON keys. ([Google Cloud Documentation](https://docs.cloud.google.com/dotnet/docs/getting-started/deploying-to-gke-using-github-actions?utm_source=chatgpt.com))

You need:

* a Google service account  
* a workload identity pool/provider  
* IAM bindings so GitHub Actions can impersonate the service account  
* GitHub repo secrets/variables with the provider and service account email

## **Create service account**

gcloud iam service-accounts create github-actions-gke \\  
  \--display-name="GitHub Actions GKE deployer"

## **Grant roles**

For this lab, keep it simple:

gcloud projects add-iam-policy-binding $PROJECT\_ID \\  
  \--member="serviceAccount:github-actions-gke@$PROJECT\_ID.iam.gserviceaccount.com" \\  
  \--role="roles/container.developer"

gcloud projects add-iam-policy-binding $PROJECT\_ID \\  
  \--member="serviceAccount:github-actions-gke@$PROJECT\_ID.iam.gserviceaccount.com" \\  
  \--role="roles/artifactregistry.writer"

gcloud projects add-iam-policy-binding $PROJECT\_ID \\  
  \--member="serviceAccount:github-actions-gke@$PROJECT\_ID.iam.gserviceaccount.com" \\  
  \--role="roles/container.clusterViewer"

You’ll also need Workload Identity Federation setup from Google’s documented GitHub Actions pattern. The exact setup is a few steps, so use the official guide while doing it once. The key idea you should be able to explain in interview is: **GitHub issues an OIDC token, Google trusts that identity provider, and GitHub Actions impersonates a service account without storing a key.** ([Google Cloud Documentation](https://docs.cloud.google.com/dotnet/docs/getting-started/deploying-to-gke-using-github-actions?utm_source=chatgpt.com))

## **In GitHub repo settings, add:**

**Secrets / Variables**

* `GCP_PROJECT_ID`  
* `GCP_WIF_PROVIDER`  
* `GCP_SERVICE_ACCOUNT`  
* `GKE_CLUSTER`  
* `GKE_ZONE`  
* `ARTIFACT_REGISTRY_REGION`  
* `ARTIFACT_REGISTRY_REPO`

---

# **5\) Create the GitHub Actions workflow**

## **`.github/workflows/deploy.yml`**

name: Build and Deploy to GKE

on:  
  push:  
    branches: \[ "main" \]

permissions:  
  contents: read  
  id-token: write

env:  
  PROJECT\_ID: ${{ vars.GCP\_PROJECT\_ID }}  
  GAR\_REGION: ${{ vars.ARTIFACT\_REGISTRY\_REGION }}  
  GAR\_REPO: ${{ vars.ARTIFACT\_REGISTRY\_REPO }}  
  CLUSTER\_NAME: ${{ vars.GKE\_CLUSTER }}  
  CLUSTER\_ZONE: ${{ vars.GKE\_ZONE }}  
  IMAGE\_NAME: hello-gke

jobs:  
  build-and-deploy:  
    runs-on: ubuntu-latest

    steps:  
      \- name: Checkout  
        uses: actions/checkout@v4

      \- name: Authenticate to Google Cloud  
        uses: google-github-actions/auth@v2  
        with:  
          workload\_identity\_provider: ${{ vars.GCP\_WIF\_PROVIDER }}  
          service\_account: ${{ vars.GCP\_SERVICE\_ACCOUNT }}

      \- name: Set up gcloud  
        uses: google-github-actions/setup-gcloud@v2

      \- name: Configure Docker auth for Artifact Registry  
        run: gcloud auth configure-docker $GAR\_REGION-docker.pkg.dev \--quiet

      \- name: Build image  
        run: |  
          docker build \-t $GAR\_REGION-docker.pkg.dev/$PROJECT\_ID/$GAR\_REPO/$IMAGE\_NAME:${{ github.sha }} ./app

      \- name: Push image  
        run: |  
          docker push $GAR\_REGION-docker.pkg.dev/$PROJECT\_ID/$GAR\_REPO/$IMAGE\_NAME:${{ github.sha }}

      \- name: Get GKE credentials  
        run: |  
          gcloud container clusters get-credentials $CLUSTER\_NAME \--zone $CLUSTER\_ZONE \--project $PROJECT\_ID

      \- name: Render deployment image  
        run: |  
          sed "s|IMAGE\_PLACEHOLDER|$GAR\_REGION-docker.pkg.dev/$PROJECT\_ID/$GAR\_REPO/$IMAGE\_NAME:${{ github.sha }}|g" \\  
          k8s/deployment.yaml \> k8s/deployment-rendered.yaml

      \- name: Deploy to GKE  
        run: |  
          kubectl apply \-f k8s/deployment-rendered.yaml  
          kubectl apply \-f k8s/service.yaml  
          kubectl rollout status deployment/hello-gke \--timeout=120s

      \- name: Show service  
        run: kubectl get svc hello-gke-service

GitHub Actions supports build-and-push workflows and uses contexts like `github.sha`, while Google’s GitHub Actions guidance supports authenticating to GCP and deploying to GKE. ([GitHub Docs](https://docs.github.com/actions/guides/publishing-docker-images?utm_source=chatgpt.com))

---

# **6\) First successful run**

Push to `main`.

Then check:

kubectl get pods  
kubectl get svc hello-gke-service

When the external IP appears, open it in the browser. GKE’s documented deploy flow includes creating the workload, checking the deployment, and exposing it. ([Google Cloud Documentation](https://docs.cloud.google.com/kubernetes-engine/docs/deploy-app-cluster?utm_source=chatgpt.com))

---

# **7\) Now do the interview-grade practice: break it and fix it**

This is the important part.

## **Break 1: Wrong `targetPort`**

Change `service.yaml`:

targetPort: 3000

Push again.

### **What happens**

Pods are healthy, but app is unreachable because the service is forwarding to the wrong container port.

### **How to debug**

kubectl get svc  
kubectl describe svc hello-gke-service  
kubectl get endpoints hello-gke-service  
kubectl get pods  
kubectl logs deploy/hello-gke

### **What to say in interview**

I would validate the traffic path layer by layer: pod health, container port, service port, targetPort, endpoints, and external exposure.

---

## **Break 2: Wrong selector labels**

Change `service.yaml`:

selector:  
  app: wrong-label

### **What happens**

The service has no endpoints.

### **How to debug**

kubectl get svc  
kubectl get endpoints hello-gke-service  
kubectl get pods \--show-labels  
kubectl describe svc hello-gke-service

### **Interview answer**

If endpoints are empty, I immediately suspect a label mismatch between the service selector and the pod template labels.

---

## **Break 3: Bad image tag or bad image**

Change the Dockerfile to something broken, or point deployment to a non-existent image.

### **What happens**

Pods go `ImagePullBackOff` or fail rollout.

### **How to debug**

kubectl get pods  
kubectl describe pod \<pod-name\>  
kubectl rollout status deployment/hello-gke

### **Interview answer**

I’d check pod events first because `describe pod` usually shows image pull failures, registry auth issues, or bad tags very clearly.

---

## **Break 4: Broken health endpoint**

Change `server.js` so `/health` returns 500\.

### **What happens**

Readiness/liveness probes fail, rollout may hang or pods restart.

### **How to debug**

kubectl get pods  
kubectl describe pod \<pod-name\>  
kubectl logs \<pod-name\>  
kubectl rollout status deployment/hello-gke

### **Interview answer**

A running pod is not enough. I always verify readiness and liveness because traffic only reaches ready pods, and liveness affects restarts.

---

# **8\) The exact practice plan for the next 2 days**

## **Today**

Build the repo and get one successful deployment working.

Do these in order:

1. create cluster  
2. create Artifact Registry repo  
3. set up GitHub Actions auth  
4. push app  
5. verify service external IP  
6. screenshot or note every command you used

## **Tomorrow**

Run the 4 failure drills above.

For each one, write down:

* symptom  
* command you ran first  
* root cause  
* fix

That turns this from “I know Kubernetes” into “I debug Kubernetes in a structured way.”

---

# **9\) What to say if they ask, “Explain this pipeline”**

Use this:

I created a GitHub Actions pipeline triggered on pushes to the main branch. The workflow authenticated to Google Cloud using Workload Identity Federation, built the application Docker image, pushed it to Artifact Registry, fetched GKE cluster credentials, and deployed the updated Kubernetes manifests with the new image tag. I exposed the application using a LoadBalancer service and validated rollout health using Kubernetes rollout status. I also practiced failure scenarios like wrong ports, label mismatches, bad images, and failing health checks so I could debug the deployment path end to end. ([GitHub Docs](https://docs.github.com/en/actions/get-started/quickstart?utm_source=chatgpt.com))

---

# **10\) What they may ask you from this lab**

Here are the best short answers to practice:

**Why GitHub Actions?**  
Because it integrates closely with the source repo and can automate build/test/deploy on pushes and pull requests. ([GitHub Docs](https://docs.github.com/en/actions/get-started/quickstart?utm_source=chatgpt.com))

**Why Artifact Registry?**  
Because it’s GCP’s container artifact store and integrates cleanly with GKE image pulls. ([Google Cloud Documentation](https://docs.cloud.google.com/artifact-registry/docs/integrate-gke?utm_source=chatgpt.com))

**Why Workload Identity Federation?**  
Because it avoids storing long-lived service account keys and uses short-lived identity-based authentication from GitHub Actions. ([Google Cloud Documentation](https://docs.cloud.google.com/dotnet/docs/getting-started/deploying-to-gke-using-github-actions?utm_source=chatgpt.com))

**Why `LoadBalancer` service?**  
Because it gives you a simple external entry point backed by Google Cloud load balancing. ([Google Cloud Documentation](https://docs.cloud.google.com/kubernetes-engine/docs/concepts/service-load-balancer?utm_source=chatgpt.com))

**Why use `github.sha` for image tags?**  
Because it gives traceability from deployed image back to the exact commit that produced it. GitHub Actions exposes commit metadata through workflow contexts. ([GitHub Docs](https://docs.github.com/en/actions/reference/workflows-and-actions/contexts?utm_source=chatgpt.com))

---

If you want, I can turn this into a **copy-paste ready checklist with exact commands and expected outputs**, so you can execute it without thinking.

