# Google Cloud Platform (GCP): Core Concepts and Fundamentals

## 1. Core Concept

**Google Cloud Platform** is Google's public cloud offering for compute, storage, networking, and data services:

1. **Compute**: Virtual machines (Compute Engine), containers (GKE, Cloud Run), serverless (Cloud Functions)
2. **Storage**: Object storage (Cloud Storage), databases (Cloud SQL, Firestore, Datastore)
3. **Networking**: VPCs, Cloud Load Balancing, Cloud CDN, Cloud Interconnect
4. **Data & AI**: BigQuery, Dataflow, Vertex AI, Pub/Sub
5. **Management**: Cloud Build, Cloud Deployment Manager, Terraform
6. **Security**: Cloud IAM, Cloud KMS, VPC Service Controls

### Why GCP for Enterprise

```
Traditional Infrastructure:
├─ On-premise data centers
├─ Capital expenditure (hardware)
├─ Ops team manages everything
├─ Slow to scale
└─ Poor disaster recovery

GCP (Cloud):
├─ Global infrastructure (30+ regions)
├─ Operational expenditure (pay-as-you-go)
├─ Google manages infrastructure
├─ Auto-scaling in seconds
├─ Built-in redundancy & recovery
└─ Focus on applications!
```

---

## 2. GCP Architecture

### GCP Organizational Structure

```
Organization (top-level)
├─ Folder: prod
│  ├─ Project: prod-platform
│  │  ├─ Compute Engine (VMs)
│  │  ├─ GKE (Kubernetes)
│  │  ├─ Cloud SQL (Database)
│  │  └─ Cloud Storage (Files)
│  │
│  └─ Project: prod-analytics
│     ├─ BigQuery (Data warehouse)
│     └─ Cloud Pub/Sub (Messaging)
│
├─ Folder: staging
│  └─ Project: staging-platform
│
└─ Folder: dev
   └─ Project: dev-platform

# Key Concepts:
├─ Organization: Highest billing unit
├─ Folder: Organizational grouping (no billing)
├─ Project: Billing & resource container
├─ Resources: VMs, databases, etc (in projects)
└─ IAM: Access control at any level
```

### GCP Resource Hierarchy & IAM

```
IAM Policy Inheritance:

Organization (Owner can delete org)
    ↓
Folder (Project Admin can manage)
    ↓
Project (Resource Admin can manage)
    ↓
Resource (lowest scope: VM, database, etc)

Permissions flow downward:
├─ Organization Editor
│  └─ Can edit everything in org
│
├─ Folder Editor
│  └─ Can edit everything in folder
│     (but not above folder)
│
├─ Project Editor
│  └─ Can edit resources in project
│
└─ Resource-level roles
   └─ Can edit specific resource
```

### GCP Regions & Zones

```
Global Infrastructure:

├─ Regions (30+)
│  ├─ us-central1 (Iowa)
│  ├─ us-east1 (South Carolina)
│  ├─ europe-west1 (Belgium)
│  ├─ asia-east1 (Taiwan)
│  └─ etc
│
└─ Zones (within each region)
   ├─ us-central1-a
   ├─ us-central1-b
   ├─ us-central1-c
   └─ us-central1-d

Benefits:
├─ High availability (replicate across zones)
├─ Latency (choose region near users)
├─ Compliance (data residency)
└─ Cost optimization
```

---

## 3. Core GCP Services

### Compute Options (When to Use)

```
Compute Engine:
├─ Use when: Full VM control needed
├─ Example: Database servers, legacy apps
├─ Cost: Pay per minute
├─ Management: You manage OS patches
└─ Scaling: Manual or Managed Instance Groups

GKE (Kubernetes):
├─ Use when: Microservices, containers
├─ Example: Distributed systems, APIs
├─ Cost: Per container/Pod
├─ Management: Google manages K8s control plane
└─ Scaling: Automatic (Horizontal Pod Autoscaler)

Cloud Run:
├─ Use when: Serverless, event-driven
├─ Example: API endpoints, background jobs
├─ Cost: Per request (free tier: 2M reqs/month)
├─ Management: Just deploy containers
└─ Scaling: Zero to millions instantly

Cloud Functions:
├─ Use when: Simple code (not containers)
├─ Example: Webhooks, scheduled tasks
├─ Cost: Per 100ms of execution
├─ Management: No infrastructure
└─ Scaling: Automatic

Decision Tree:
┌─ Full control needed? ──Yes──→ Compute Engine
│
├─ Containers? ──Yes──→ Yes–─ Many replicas? ──Yes──→ GKE
│                      │
│                      └─ Few replicas? ──→ Cloud Run
│
└─ Just code? ──Yes──→ Cloud Functions
```

### Storage Options

```
Cloud Storage:
├─ Use: Object storage (files, images, backups)
├─ Price: $0.020/GB/month (us-multi-region)
├─ Access: HTTP API, gsutil CLI, SDK
└─ Example: Store app backups, user uploads

Cloud SQL:
├─ Use: Relational database (MySQL, PostgreSQL)
├─ Price: ~$9/month (small instance)
├─ Managed: Google patches, backups, replication
└─ Example: Application database

Firestore:
├─ Use: NoSQL document database
├─ Price: $0.06/100K reads (generous free tier)
├─ Managed: Firebase integration, real-time sync
└─ Example: Mobile apps, real-time collaboration

BigQuery:
├─ Use: Data warehouse (OLAP, analytics)
├─ Price: $6.25/TB scanned (free: 1TB/month)
├─ Speed: Query results in seconds (TB scale)
└─ Example: Company analytics, dashboards
```

---

## 4. Real-World GCP Workflows

### Workflow 1: Deploying a Web Application

```
Architecture:
┌──────────────────────────────────────┐
│ Cloud Load Balancer (Global)         │
│ ├─ Route from internet               │
│ └─ Auto-scale based on load          │
├──────────────────┬───────────────────┤
│ GKE Cluster      │                   │
│ prod-us-central  │ prod-us-east      │
│ (3 replicas)     │ (3 replicas)      │
├──────────────────┴───────────────────┤
│ Cloud SQL         │ Cloud Storage    │
│ (PostgreSQL)      │ (User uploads)   │
└──────────────────────────────────────┘

Step 1: Create GCP Project
$ gcloud projects create my-app --name "My App"

Step 2: Create Kubernetes cluster
$ gcloud container clusters create prod-us-central \
    --region us-central1 \
    --num-nodes 3 \
    --machine-type n1-standard-2

Step 3: Create Cloud SQL database
$ gcloud sql instances create app-db \
    --database-version POSTGRES_14 \
    --tier db-f1-micro \
    --region us-central1

Step 4: Deploy application to GKE
$ kubectl create deployment myapp --image=gcr.io/my-app:v1
$ kubectl expose deployment myapp --port=80 --type=LoadBalancer

Step 5: Create Load Balancer
$ gcloud compute backend-services create app-backend \
    --global \
    --health-checks=healthcheck

Step 6: Monitor
$ gcloud monitoring dashboards create --config-from-file=dashboard.json

Result: App deployed, auto-scaling, globally distributed!
```

### Workflow 2: Data Pipeline with Pub/Sub & BigQuery

```
Architecture:
┌──────────────────────────────────────┐
│ Application Servers (GKE)            │
│ ├─ Generate events (user clicks)     │
│ └─ Publish to Pub/Sub                │
├──────────────────────────────────────┤
│ Cloud Pub/Sub (Message Queue)        │
│ ├─ Decouple producers from consumers │
│ └─ 10ms latency guarantee            │
├──────────────────────────────────────┤
│ Cloud Dataflow (ETL Pipeline)        │
│ ├─ Transform events                  │
│ └─ Filter, aggregate, enrich         │
├──────────────────────────────────────┤
│ BigQuery (Data Warehouse)            │
│ ├─ Store processed events            │
│ └─ Analytics queries                 │
└──────────────────────────────────────┘

Code Example:

# Publisher (sends events)
from google.cloud import pubsub_v1

publisher = pubsub_v1.PublisherClient()
topic_path = publisher.topic_path(project, "user-events")

for event in events:
    publisher.publish(topic_path, data=json.dumps(event).encode())

# Subscriber (processes events)
from google.cloud import pubsub_v1

subscriber = pubsub_v1.SubscriberClient()
subscription_path = subscriber.subscription_path(project, "analytics-sub")

def callback(message):
    print(f"Received: {message.data.decode()}")
    message.ack()

subscriber.subscribe(subscription_path, callback=callback)

# Dataflow (Apache Beam)
import apache_beam as beam
from apache_beam.options.pipeline_options import PipelineOptions

options = PipelineOptions()
with beam.Pipeline(options=options) as p:
    (p 
     | 'Read from Pub/Sub' >> beam.io.ReadFromPubSub(topic=pubsub_topic)
     | 'Parse JSON' >> beam.Map(json.loads)
     | 'Filter' >> beam.Filter(lambda x: x['type'] == 'click')
     | 'Write to BigQuery' >> beam.io.WriteToBigQuery(table=bq_table))

# BigQuery (Analytics Query)
SELECT 
    DATE(timestamp) as date,
    COUNT(*) as clicks,
    COUNT(DISTINCT user_id) as users
FROM `project.dataset.events`
GROUP BY date
ORDER BY date DESC;

Result: Real-time analytics without managing infrastructure!
```

### Workflow 3: CI/CD with Cloud Build

```yaml
# cloudbuild.yaml
steps:
  # Step 1: Build Docker image from source
  - name: 'gcr.io/cloud-builders/docker'
    args: 
      - 'build'
      - '-t'
      - 'gcr.io/$PROJECT_ID/myapp:$SHORT_SHA'
      - '.'

  # Step 2: Push to Container Registry
  - name: 'gcr.io/cloud-builders/docker'
    args:
      - 'push'
      - 'gcr.io/$PROJECT_ID/myapp:$SHORT_SHA'

  # Step 3: Deploy to GKE
  - name: 'gcr.io/cloud-builders/gke-deploy'
    args:
      - run
      - --filename=k8s/
      - --image=gcr.io/$PROJECT_ID/myapp:$SHORT_SHA
      - --location=us-central1
      - --cluster=prod-cluster

  # Step 4: Smoke tests
  - name: 'gcr.io/cloud-builders/kubectl'
    args:
      - 'rollout'
      - 'status'
      - 'deployment/myapp'
      - '-n'
      - 'production'
    env:
      - 'CLOUDSDK_COMPUTE_REGION=us-central1'
      - 'CLOUDSDK_CONTAINER_CLUSTER=prod-cluster'

# Trigger on git push
onFailure:
  - name: 'gcr.io/cloud-builders/gke-deploy'
    args:
      - 'run'
      - '--filename=k8s/'
      - '--image=gcr.io/$PROJECT_ID/myapp:previous'
      - '--rollback'

images:
  - 'gcr.io/$PROJECT_ID/myapp:$SHORT_SHA'
```

---

## 5. Common Mistakes

### Mistake 1: Wrong Region Choice

```bash
# ❌ WRONG
$ gcloud compute instances create myvm \
    --zone us-central1-a
# If zone goes down → all users affected!

# ✅ CORRECT
# Replicate across multiple zones
$ gcloud container clusters create prod \
    --region us-central1 \
    --num-nodes 3
# Kubernetes spreads pods across 3 zones automatically!

# Or multi-region
$ gcloud container clusters create prod-us
    --region us-central1
$ gcloud container clusters create prod-eu
    --region europe-west1
# Traffic routed to closest region
```

### Mistake 2: Not Using Managed Services

```bash
# ❌ WRONG (Managing yourself)
# Deploy your own PostgreSQL on Compute Engine
$ gcloud compute instances create mydb --machine-type n1-standard-4
# You manage: Patches, backups, replication, updates

# ✅ CORRECT (Managed service)
$ gcloud sql instances create mydb \
    --database-version POSTGRES_14 \
    --tier db-f1-micro
# Google manages: Patches, backups, replication, high availability

# Cost difference: ~30% cheaper with managed!
```

### Mistake 3: Over-provisioning Resources

```bash
# ❌ WRONG
$ gcloud compute instances create myvm \
    --machine-type n1-highmem-16 \
    --zone us-central1-a
# 16 CPUs, 104GB RAM = $0.66/hour = $480+/month
# But app uses 10% of resources!

# ✅ CORRECT
# Start small, monitor, scale
$ gcloud compute instances create myvm \
    --machine-type n1-standard-1 \
    --zone us-central1-a
# 1 CPU, 3.75GB RAM = $0.03/hour = $25/month

# Use metrics-based autoscaling
$ gcloud compute instance-groups managed set-autoscaling prod-ig \
    --max-num-replicas 10 \
    --min-num-replicas 2 \
    --target-cpu-utilization 0.7
```

### Mistake 4: No IAM Role Separation

```bash
# ❌ WRONG
# Give everyone "Editor" role
$ gcloud projects add-iam-policy-binding myproject \
    --member user:engineer@company.com \
    --role roles/editor
# Can delete everything!

# ✅ CORRECT (Principle of least privilege)
# Backend engineers: Can deploy backend only
$ gcloud projects add-iam-policy-binding myproject \
    --member group:backend-team@company.com \
    --role roles/container.developer

# Frontend engineers: Can deploy frontend only
$ gcloud projects add-iam-policy-binding myproject \
    --member group:frontend-team@company.com \
    --role roles/container.developer

# DBAs: Can manage Cloud SQL only
$ gcloud projects add-iam-policy-binding myproject \
    --member group:dba-team@company.com \
    --role roles/cloudsql.admin
```

### Mistake 5: Not Using Cloud Logging

```bash
# ❌ WRONG
# Logs scattered across:
# ├─ Local application logs
# ├─ Kubernetes logs
# ├─ Database logs
# └─ Hard to correlate!

# ✅ CORRECT (Unified logging)
# All logs → Cloud Logging

# Application sends logs
import logging
from google.cloud import logging as cloud_logging

client = cloud_logging.Client()
client.setup_logging()
logging.info("Application event", extra={"user_id": 123})

# Query all logs
$ gcloud logging read "resource.type=gke_container AND severity=ERROR"

# Create alerts
$ gcloud alpha monitoring policies create \
    --display-name="High error rate" \
    --condition="error_count > 100"
```

---

## 6. Debugging GCP Issues

### Problem: Deployment Timeout

```bash
# Error: Deployment stuck in "Pending"

# Debug steps:
1. Check pod status
   $ kubectl get pods -n default
   # Shows: "Pending", "ImagePullBackOff", etc

2. Describe pod for details
   $ kubectl describe pod myapp-xxxxx
   # Shows: "Insufficient CPU", "Image not found", etc

3. Check container image
   $ gcloud container images list
   $ gcloud container images describe gcr.io/project/myapp:v1

4. Check GKE cluster resources
   $ gcloud container clusters describe prod-cluster \
       --region us-central1

# Solution:
# Usually: Out of resources or image not found
# Fix:
#   - Add more nodes to cluster
#   - Reduce resource requests
#   - Fix image URI
```

### Problem: High Costs

```bash
# Investigating unexpected bills:

1. Check estimated costs
   $ gcloud billing accounts describe <ACCOUNT_ID> \
       --format='value(displayName)'

2. Detailed breakdown
   $ gcloud billing projects describe myproject

3. Find resource hogs
   $ gcloud compute instances list --format='table(name, MACHINE_TYPE, STATUS)'
   $ gcloud sql instances list

4. Check idle resources
   $ gcloud compute disks list --filter="users:*(users='')"
   # Unattached disks still cost money!

# Common cost wasters:
├─ Idle VMs (stopped = still $0.02/month for disk)
├─ Unattached persistent disks
├─ Large data transfers (egress = $0.12/GB)
├─ BigQuery storage (unused tables)
└─ Over-provisioned resources

# Solutions:
├─ Delete unused resources
├─ Use committed discounts (25-30% savings)
├─ Use preemptible VMs (70% discount, can interrupt)
├─ Enable BigQuery table expiration
└─ Auto-scale down during low usage
```

---

## 7. Interview Questions (Scenario-Based)

### **Q1**: Design a global web application on GCP that handles 100K requests/second with 99.99% availability.

**Answer**:
```
Architecture:

Global Traffic:
├─ Cloud CDN (edge caching)
│  ├─ Cache static assets at 200+ locations
│  ├─ 10-100x faster for users
│  └─ Only cache hits reach origin
│
├─ Cloud Load Balancing (global)
│  ├─ Route traffic to nearest region
│  ├─ Health-check backends every 10s
│  └─ Failover if region down
│
└─ Multiple Regional Clusters:
   ├─ GKE in us-central1
   │  ├─ 5-10 nodes per region
   │  ├─ Auto-scale based on CPU
   │  └─ Pod disruption budgets (maintain SLA)
   │
   ├─ GKE in europe-west1
   │
   └─ GKE in asia-east1

Database:
├─ Cloud SQL with high availability
│  ├─ Primary in us-central1
│  ├─ Replica in us-east1 (failover)
│  └─ Read replicas in eu, asia
│
└─ Firestore (real-time, globally replicated)

Messaging:
├─ Cloud Pub/Sub
│  ├─ Decouple services
│  ├─ Auto-scale subscribers
│  └─ At least once delivery

Monitoring:
├─ Cloud Monitoring dashboards
├─ Alerting on error rate >0.01%
└─ Auto-scaling policies
   ├─ CPU > 70% → Add pods
   ├─ CPU < 30% → Remove pods
   └─ Max 1000 pods per deployment

Result:
✅ 99.99% availability (52 mins down/year)
✅ Sub-100ms latency (global)
✅ Auto-scale to handle spikes
✅ No single point of failure
```

---

## 8. Advanced Insights

### Serverless Architecture (Cloud Run + Pub/Sub)

```
Traditional:
├─ Deploy containers to Kubernetes
├─ Pay for running containers (even idle)
├─ Min viable cluster = $200/month

Serverless:
├─ Deploy code → Cloud Run
├─ Pay per request (requests only)
├─ Free tier: 2M requests/month

Example:
┌──────────────────────────────────────┐
│ API client                           │
│ POST /process?item=123               │
├──────────────────────────────────────┤
│ Cloud Run Service                    │
│ ├─ Cold start: 100ms (first request) │
│ ├─ Warm start: 1ms (cached)          │
│ └─ Process request (max 60 mins)     │
├──────────────────────────────────────┤
│ Cloud Pub/Sub (async notification)   │
│ └─ Publish "processing complete"     │
├──────────────────────────────────────┤
│ BigQuery (store results)             │
│ └─ Query results later               │
└──────────────────────────────────────┘

Cost:
- 1M requests/month with 512MB memory
- Traditional (Kubernetes): ~$150/month
- Cloud Run: < $5/month (mostly free tier!)

Use when:
✅ Variable load (spiky traffic)
✅ Simple services (no complex infra)
✅ Cost-sensitive workloads
✅ Rapid prototyping

Avoid when:
❌ Long-running jobs (>60 min timeout)
❌ Complex infra (service mesh, etc)
❌ Predictable constant load
```

### GCP vs AWS vs Azure

```
Strength Comparison:

GCP Strengths:
├─ Data & Analytics: BigQuery, Dataflow
├─ Machine Learning: Vertex AI
├─ Cost: Discounts for sustained use
├─ Simplicity: Less complex than AWS
└─ Global network: Google's fiber backbone

AWS Strengths:
├─ Breadth: 200+ services
├─ Maturity: Oldest cloud (2006)
├─ Enterprise: Largest customer base
└─ Compute: Most instance types

Azure Strengths:
├─ Microsoft integration: Windows, SQL Server
├─ Hybrid: Azure Stack bridges on-prem/cloud
├─ Enterprise: Office 365, Dynamics integration
└─ Contracts: Volume licensing

For most: GCP > AWS > Azure
For Windows shops: Azure > GCP > AWS
For data-heavy: GCP > AWS > Azure
```

---

## 9. GCP Cost Optimization

```
Strategies:

1. Committed Use Discounts
   ├─ Reserve 1-year: 25% savings
   ├─ Reserve 3-year: 30% savings
   └─ Trade flexibility for cost

2. Preemptible VMs
   ├─ Up to 70% cheaper
   ├─ Can interrupt (30s notice)
   └─ Use for: Batch jobs, non-critical workloads

3. Autoscaling
   ├─ Only pay for used resources
   ├─ Scale down to 0 (serverless)
   └─ Reduce off-peak costs

4. Rightsizing
   ├─ Monitor actual usage
   ├─ Downsize over-provisioned resources
   └─ n1-standard-4 → n1-standard-1 saves 75%!

5. Data transfer
   ├─ Egress costs $0.12/GB
   ├─ Keep data in same region (free)
   └─ Use Cloud CDN cache (reduce egress)

Typical savings: 50-60% of compute costs
```

---

## Conclusion

GCP fundamentals enable:

✅ **Global scale**: Auto-scale to any demand  
✅ **Managed services**: Focus on code, not ops  
✅ **Cost efficiency**: Pay only for what you use  
✅ **Data & AI**: Leading data warehousing (BigQuery)  
✅ **Developer experience**: Simple APIs, great docs  

Master GCP and you can:
- ✅ Build scalable applications
- ✅ Leverage big data & ML
- ✅ Optimize cloud costs
- ✅ Deploy globally
- ✅ Maintain 99.99% availability
