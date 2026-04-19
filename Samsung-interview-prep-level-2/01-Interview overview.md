# **What This 90-Min Interview Will Look Like**

Based on your stack \+ role:

## **🧪 Likely Format**

### **⏱️ 0–15 min → Deep discussion**

* “Explain your pipeline”  
* “Explain your ELK architecture”

👉 They will **dig deeper than first round**

---

### **⏱️ 15–60 min → Practical \+ Scenario**

* Debugging problems  
* System design  
* Maybe whiteboard or laptop task

---

### **⏱️ 60–90 min → Advanced \+ Behavioral**

* Tradeoffs  
* Security  
* Decision making

---

# **🔥 What They WILL Test You On (Based on YOU)**

From your experience:

### **🔴 1\. Kubernetes (VERY DEEP)**

They will go beyond basics.

Expect:

* Pod lifecycle  
* Networking (VERY important)  
* Ingress / Services  
* Debugging

---

### **🔴 2\. CI/CD Pipeline (Your GitHub Actions)**

They will ask:

👉 “Walk me through EXACTLY what happens”

Be ready for:

* Secrets handling  
* Rollback strategy  
* Failure handling

---

### **🔴 3\. ELK \+ Logging (YOUR STRONGEST AREA 💪)**

This is your **weapon**

Expect:

* How logs flow end-to-end  
* Failure scenarios  
* Scaling issues

---

### **🔴 4\. Terraform**

* State  
* Drift  
* Modules  
* Real-world usage

---

### **🔴 5\. GCP (IMPORTANT)**

* GKE  
* IAM  
* Networking

---

# **🧠 Day-by-Day Plan (You have 2 days)**

---

# **📅 DAY 1 (TODAY) — FOUNDATION \+ DEPTH**

## **✅ 1\. MASTER YOUR OWN PROJECTS (MOST IMPORTANT)**

Prepare these 3 stories perfectly:

---

## **🔥 STORY 1: CI/CD Pipeline (GitHub Actions → GKE/EKS)**

You must explain:

* Trigger (push / PR)  
* Build Docker image  
* Push to registry  
* Deploy via Helm or kubectl  
* ArgoCD role

👉 Be ready for:

* What if deployment fails?  
* How rollback works?  
* How secrets managed?

---

## **🔥 STORY 2: ELK \+ Wazuh Pipeline**

Explain like architect:

* Fluent Bit → Logstash → Elasticsearch → Kibana  
* Where parsing happens  
* Where filtering happens  
* PII masking logic

👉 They will ask:

* What if logs stop?  
* How to scale?  
* How to reduce cost?

---

## **🔥 STORY 3: Terraform Infra**

Explain:

* How you structure code  
* Modules  
* State backend  
* Drift handling

---

## **✅ 2\. Kubernetes Deep Dive (CRITICAL)**

Focus on:

### **MUST KNOW:**

* Pod lifecycle  
* Liveness vs readiness probes  
* Service types (ClusterIP, NodePort, LoadBalancer)  
* Ingress flow

### **DEBUGGING:**

kubectl get pods  
kubectl describe pod  
kubectl logs  
kubectl exec

👉 Practice on your GCP cluster

---

## **✅ 3\. Practice 5 Real Scenarios**

I’ll give you MUST-prepare ones:

---

### **🔥 Scenario 1:**

**Pod cannot access internet**

Check:

* Network policy  
* NAT gateway  
* DNS

---

### **🔥 Scenario 2:**

**App deployed but not accessible**

Check:

* Service  
* Ingress  
* Ports

---

### **🔥 Scenario 3:**

**Logs not reaching Elasticsearch**

Check:

* Fluent Bit config  
* Network  
* Index issue

---

### **🔥 Scenario 4:**

**Terraform apply destroying resources**

Why?

* State mismatch  
* Config change

---

### **🔥 Scenario 5:**

**Pipeline passed but deployment failed**

Why?

* Image tag mismatch  
* ArgoCD sync issue

---

# **📅 DAY 2 (TOMORROW) — MOCK \+ PRACTICAL**

---

## **✅ 1\. DO HANDS-ON (VERY IMPORTANT)**

In your GCP trial:

* Create GKE cluster  
* Deploy app  
* Expose via service  
* Break it intentionally  
* Fix it

👉 This gives REAL confidence

---

## **✅ 2\. Practice Speaking (OUT LOUD)**

Say:

“First, I would check logs using kubectl logs…”

👉 This matters A LOT

---

## **✅ 3\. Do Mock Interview (WITH ME)**

Tell me:  
👉 “Start Samsung mock interview”

I’ll simulate real pressure

---

# **💥 CRITICAL: HOW TO ANSWER (THIS WILL DECIDE RESULT)**

---

## **❌ Weak Answer:**

“I think issue could be network…”

---

## **✅ Strong Answer:**

“First, I would verify pod status using `kubectl get pods`.  
Then check logs using `kubectl logs`.  
If it's a network issue, I’d verify service and DNS resolution…”

---

👉 Structured thinking \= SELECTED

---

# **🧠 Samsung-Specific Tip (VERY IMPORTANT)**

Since it's **Health Department**:

👉 Emphasize:

* Security  
* Data privacy (PII — YOU HAVE THIS 🔥)  
* Reliability

Say things like:

“Since this involves sensitive data, I ensured PII masking at Logstash layer…”

---

# **🚨 Final Warning (Mentor Mode)**

They WILL test:

* Depth ❗  
* Debugging ❗  
* Confidence ❗

They don’t care if you know everything.

They care if:  
👉 You think like engineer  
👉 You can handle production

---

# **🚀 Next Step**

We should NOT stop here.

I strongly recommend:

👉 Let me run **90-min style mock interview with you**

I will:

* Ask real Samsung-level questions  
* Interrupt you  
* Push deeper  
* Fix your answers

Just say:  
👉 **“Start mock interview”**

Let’s get you this job 💪

