# **FULL STORY: Kubernetes (GKE) Platform**

---

## **🗣️ Opening (Say this confidently)**

I have worked extensively with Kubernetes, particularly GKE, where I was responsible for deploying, managing, and troubleshooting containerized applications.

My work involved designing deployments, configuring services and ingress, managing scaling, and debugging production issues. I also integrated Kubernetes with CI/CD pipelines and logging systems to ensure reliability and observability.

---

# **🏗️ ARCHITECTURE (Explain like this)**

User Request  
    │  
    ▼  
External Load Balancer (GCP LB)  
    │  
    ▼  
Ingress Controller  
    │  
    ▼  
Kubernetes Service (ClusterIP)  
    │  
    ▼  
Pods (Application Containers)  
    │  
    ├── Logs → Fluent Bit → ELK  
    ├── Metrics → Monitoring  
    └── Health Checks → Probes

---

# **🧠 CORE CONCEPTS (INTERVIEW LEVEL)**

---

## **🟢 1\. Pod Lifecycle (VERY IMPORTANT)**

A Pod goes through multiple states:

* Pending → Scheduled → Running → Succeeded/Failed

👉 Interview line:

If a pod is stuck in Pending, I check node resources or scheduling issues.

---

## **🟡 2\. Liveness vs Readiness Probes**

### **🔹 Readiness Probe**

Determines if pod is ready to receive traffic.

### **🔹 Liveness Probe**

Determines if pod should be restarted.

👉 Strong answer:

Readiness controls traffic, liveness controls restart.

---

## **🔵 3\. Service Types**

| Type | Use |
| ----- | ----- |
| ClusterIP | internal communication |
| NodePort | expose via node IP |
| LoadBalancer | external access |

---

## **🟣 4\. Ingress Flow**

Ingress routes external traffic to services using rules.

Flow:

User → LB → Ingress → Service → Pod

---

# **🔥 DEBUGGING FLOW (THIS WILL IMPRESS THEM)**

When something breaks, say this:

---

## **🧠 Standard Debugging Approach**

kubectl get pods  
kubectl describe pod \<name\>  
kubectl logs \<pod\>  
kubectl exec \-it \<pod\> \-- sh

---

### **Step-by-step thinking:**

First, I check pod status.  
Then I describe the pod to look for events.  
Then I check logs.  
If needed, I exec into the pod to debug inside the container.

👉 THIS STRUCTURE \= GOLD

---

# **🔥 INTERVIEW QUESTIONS \+ ANSWERS**

---

## **❓ Q1: Pod is not starting. What do you do?**

### **✅ Answer:**

I check:

1. `kubectl get pods`  
2. `kubectl describe pod`  
3. Look for:  
   * image pull errors  
   * resource issues  
   * crash loops

---

## **❓ Q2: Pod is running but app not accessible**

### **✅ Answer:**

I check:

* readiness probe  
* service mapping  
* port configuration  
* ingress rules

---

## **❓ Q3: What is difference between ClusterIP and LoadBalancer?**

### **✅ Answer:**

ClusterIP is internal, while LoadBalancer exposes the service externally via cloud provider load balancer.

---

## **❓ Q4: How does scaling work?**

### **✅ Answer:**

Scaling can be done using:

* manual replicas  
* Horizontal Pod Autoscaler (CPU/memory based)

---

## **❓ Q5: What happens when pod crashes?**

### **✅ Answer:**

Kubernetes restarts the container based on restart policy. If it keeps failing, it enters CrashLoopBackOff.

---

## **❓ Q6: How does Kubernetes networking work?**

### **✅ Answer:**

Every pod gets its own IP, and pods can communicate directly. Services provide stable endpoints and load balancing.

---

## **❓ Q7: What is Ingress?**

### **✅ Answer:**

Ingress is used to route external HTTP/HTTPS traffic to services based on rules like host or path.

---

## **❓ Q8: What are common issues in Kubernetes?**

### **✅ Answer:**

* CrashLoopBackOff  
* Image pull errors  
* Service misconfiguration  
* DNS issues  
* Resource limits

---

# **🔥 REAL SCENARIOS (IMPORTANT)**

---

## **🧪 Scenario 1: Pod CrashLoopBackOff**

### **Fix:**

* check logs  
* check env vars  
* check config

---

## **🧪 Scenario 2: Service not reachable**

### **Fix:**

* check selector labels  
* check ports  
* check endpoints

---

## **🧪 Scenario 3: Ingress not working**

### **Fix:**

* check ingress rules  
* check controller  
* check DNS

---

# **🧪 🔥 HANDS-ON PRACTICE (GCP – DO THIS TODAY)**

This is **CRITICAL for your confidence**

---

# **🚀 Step 1: Create GKE Cluster**

gcloud container clusters create dev-cluster \\  
  \--zone us-central1-a \\  
  \--num-nodes 2

---

# **🚀 Step 2: Connect**

gcloud container clusters get-credentials dev-cluster \--zone us-central1-a

---

# **🚀 Step 3: Deploy App**

apiVersion: apps/v1  
kind: Deployment  
metadata:  
  name: my-app  
spec:  
  replicas: 2  
  selector:  
    matchLabels:  
      app: my-app  
  template:  
    metadata:  
      labels:  
        app: my-app  
    spec:  
      containers:  
      \- name: app  
        image: nginx  
        ports:  
        \- containerPort: 80

Apply:

kubectl apply \-f deployment.yaml

---

# **🚀 Step 4: Create Service**

apiVersion: v1  
kind: Service  
metadata:  
  name: my-service  
spec:  
  type: LoadBalancer  
  selector:  
    app: my-app  
  ports:  
  \- port: 80  
    targetPort: 80

kubectl apply \-f service.yaml

---

# **🚀 Step 5: Test**

kubectl get svc

👉 Open external IP

---

# **🔥 BREAK THINGS (IMPORTANT)**

---

## **❌ Break 1: Wrong port**

Change:

targetPort: 3000

👉 Debug it

---

## **❌ Break 2: Wrong labels**

Change:

app: wrong-label

👉 Service won’t connect

---

## **❌ Break 3: Crash pod**

Use bad image:

image: nginx:invalid

👉 Debug CrashLoop

---

# **🎯 FINAL POWER STATEMENT**

My approach to Kubernetes is not just deploying applications, but understanding how each layer works and being able to debug issues systematically across pods, services, networking, and infrastructure.

---

# **💥 Mentor Advice**

If you do:

* Hands-on practice ✅  
* Speak structured debugging steps ✅  
* Explain architecture clearly ✅

👉 You will **stand out strongly**

---

# **🚀 NEXT STEP (VERY IMPORTANT)**

Let’s simulate real interview:

👉 I will ask:

* “Pod not accessible — debug”  
* “Design scalable Kubernetes system”  
* “Fix broken deployment”

Just say:  
👉 **“Start Kubernetes mock interview”**

Let’s get you selected 💪

