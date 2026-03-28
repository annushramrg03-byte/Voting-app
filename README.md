# ☁️ Cloud-Native Web Voting Application with Kubernetes

A cloud-native voting application deployed on **AWS EKS**, allowing users to vote for their favourite programming language from: **C#, Python, JavaScript, Go, Java, and NodeJS**.

---

## 🧱 Technical Stack

| Layer | Technology |
|---|---|
| Frontend | React + JavaScript |
| Backend API | Go (Golang) |
| Database | MongoDB (Replica Set) |
| Orchestration | Kubernetes (AWS EKS) |
| Infrastructure | AWS EC2, IAM, EKS |

---

## 📦 Kubernetes Resources Used

| Resource | Purpose |
|---|---|
| **Namespace** | Isolates app components (`cloudchamp`) |
| **Secret** | Stores MongoDB credentials securely |
| **Deployment** | Manages API and Frontend pods |
| **Service** | Exposes API and Frontend via LoadBalancer |
| **StatefulSet** | Manages MongoDB pods with stable identities |
| **PersistentVolume / PVC** | Provides persistent storage for MongoDB |

---

## 🚀 Steps to Deploy

### Step 1: Launch EC2 Instance (Management Machine)

- Launch an **EC2 T3.micro** instance with **Linux OS**
- In **Advanced Settings**, attach the IAM role: `my-management`

> **Why?** This EC2 acts as your control station to run all `kubectl` and `aws` commands. The IAM role gives it permission to interact with AWS services.

---

### Step 2: Install kubectl

```bash
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.24.11/2023-03-17/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo cp ./kubectl /usr/local/bin
export PATH=/usr/local/bin:$PATH
```

> **Why?** `kubectl` is the command-line tool to manage your Kubernetes cluster — deploying apps, checking node status, viewing logs, etc.

---

### Step 3: Install AWS CLI

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

> **Why?** AWS CLI lets you interact with AWS services from the terminal. You'll use it to connect `kubectl` to your EKS cluster.

---

### Step 4: Install Git

```bash
sudo yum install git -y
git --version
```

> **Why?** Git is needed to clone the application source code and Kubernetes manifest files from GitHub.

---

### Step 5: Create IAM Roles in AWS Console

Go to **IAM → Roles** and create the following roles with their policies:

**`my-management`** (for EC2 instance):
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "eks:DescribeCluster",
      "eks:ListClusters",
      "eks:DescribeNodegroup",
      "eks:ListNodegroups",
      "eks:ListUpdates",
      "eks:AccessKubernetesApi"
    ],
    "Resource": "*"
  }]
}
```

**`my-node`** (for EKS Worker Nodes) — attach the following AWS managed policies:
- `AmazonEKSWorkerNodePolicy`
- `AmazonEC2ContainerRegistryReadOnly`
- `AmazonEKS_CNI_Policy`

> **Why?** Without IAM roles, EC2 and EKS nodes cannot communicate with AWS services. Roles grant the right permissions securely without hardcoding credentials.

---

### Step 6: Create EKS Cluster

1. Go to **AWS Console → EKS → Create Cluster**
2. Give it a name (e.g., `mycluster`)
3. ⚠️ **Turn OFF "EKS Auto Mode"** — this prevents AWS from auto-creating instances so you can control the node group manually
4. Select the cluster IAM role and complete creation

> **Why?** EKS is AWS's managed Kubernetes service. It handles the control plane (master nodes) for you, so you only manage worker nodes.

---

### Step 7: Add Node Group

1. After cluster is created, go to **Compute → Add Node Group**
2. Attach the `my-node` IAM role
3. Choose instance type: **t2.medium** (2 nodes recommended)
4. Complete and wait for nodes to become **Active**

> **Why?** Worker nodes are the EC2 instances that actually run your application containers. A Node Group is a managed set of these EC2 instances.

---

### Step 8: Connect kubectl to Your EKS Cluster

```bash
aws eks update-kubeconfig --name mycluster --region ap-south-1
```

Verify nodes are ready:
```bash
kubectl get nodes
```

You should see nodes in **Ready** state.

> **Why?** This command sets up `~/.kube/config` so `kubectl` knows which cluster to talk to and how to authenticate with it.

> ⚠️ If you get `"You must be logged in to the server (Unauthorized)"` error, refer: https://repost.aws/knowledge-center/eks-api-server-unauthorized-error

---

### Step 9: Clone the GitHub Repository

```bash
git clone https://github.com/annushramrg03-byte/K8s-voting-app
cd K8s-voting-app
```

> **Why?** The Kubernetes YAML manifest files (deployments, services, secrets, etc.) are in this repo. You need them to deploy the app.

---

### Step 10: Create Namespace

```bash
kubectl create ns cloudchamp
kubectl config set-context --current --namespace cloudchamp
```

> **Why?** Namespaces logically separate resources in Kubernetes. Using `cloudchamp` groups all app resources together, making them easier to manage and clean up.

---

### Step 11: Deploy MongoDB StatefulSet

```bash
kubectl apply -f mongo-statefulset.yaml
kubectl apply -f mongo-service.yaml
```

Verify DNS records are created for each MongoDB pod:
```bash
kubectl run --rm utils -it --image praqma/network-multitool -- bash
```
Inside the pod run:
```bash
for i in {0..2}; do nslookup mongo-$i.mongo; done
exit
```

> **Why?** MongoDB is deployed as a StatefulSet (not a regular Deployment) because each pod needs a stable, unique identity (mongo-0, mongo-1, mongo-2) for the replica set to work. The DNS check confirms each pod is reachable.

---

### Step 12: Initialize MongoDB Replica Set

Run the following command on `mongo-0`:
```bash
cat << EOF | kubectl exec -it mongo-0 -- mongo
rs.initiate();
sleep(2000);
rs.add("mongo-1.mongo:27017");
sleep(2000);
rs.add("mongo-2.mongo:27017");
sleep(2000);
cfg = rs.conf();
cfg.members[0].host = "mongo-0.mongo:27017";
rs.reconfig(cfg, {force: true});
sleep(5000);
EOF
```

> ⏳ Wait 10–15 seconds until you see the `bye` message.

Confirm replica set status:
```bash
kubectl exec -it mongo-0 -- mongo --eval "rs.status()" | grep "PRIMARY\|SECONDARY"
```

> **Why?** A Replica Set replicates data across 3 MongoDB instances for **high availability**. If one instance fails, the others keep the app running.

---

### Step 13: Seed the Database

> ⚠️ Note: Use `langdb` not `langdb()` as shown in some videos.

```bash
cat << EOF | kubectl exec -it mongo-0 -- mongo
use langdb;
db.languages.insert({"name" : "csharp", "codedetail" : { "usecase" : "system, web, server-side", "rank" : 5, "compiled" : false, "homepage" : "https://dotnet.microsoft.com/learn/csharp", "download" : "https://dotnet.microsoft.com/download/", "votes" : 0}});
db.languages.insert({"name" : "python", "codedetail" : { "usecase" : "system, web, server-side", "rank" : 3, "script" : false, "homepage" : "https://www.python.org/", "download" : "https://www.python.org/downloads/", "votes" : 0}});
db.languages.insert({"name" : "javascript", "codedetail" : { "usecase" : "web, client-side", "rank" : 7, "script" : false, "homepage" : "https://en.wikipedia.org/wiki/JavaScript", "download" : "n/a", "votes" : 0}});
db.languages.insert({"name" : "go", "codedetail" : { "usecase" : "system, web, server-side", "rank" : 12, "compiled" : true, "homepage" : "https://golang.org", "download" : "https://golang.org/dl/", "votes" : 0}});
db.languages.insert({"name" : "java", "codedetail" : { "usecase" : "system, web, server-side", "rank" : 1, "compiled" : true, "homepage" : "https://www.java.com/en/", "download" : "https://www.java.com/en/download/", "votes" : 0}});
db.languages.insert({"name" : "nodejs", "codedetail" : { "usecase" : "system, web, server-side", "rank" : 20, "script" : false, "homepage" : "https://nodejs.org/en/", "download" : "https://nodejs.org/en/download/", "votes" : 0}});
db.languages.find().pretty();
EOF
```

> **Why?** This seeds the database with the 6 programming languages that users will vote on. Each starts with 0 votes.

---

### Step 14: Apply MongoDB Secret

```bash
kubectl apply -f mongo-secret.yaml
```

> **Why?** Kubernetes Secrets store sensitive data like database credentials securely, so they aren't hardcoded in deployment files.

---

### Step 15: Deploy the API

```bash
kubectl apply -f api-deployment.yaml
```

Expose the API via a LoadBalancer:
```bash
kubectl expose deploy api \
  --name=api \
  --type=LoadBalancer \
  --port=80 \
  --target-port=8080
```

Wait for DNS to propagate and confirm the API is working:
```bash
{
API_ELB_PUBLIC_FQDN=$(kubectl get svc api -ojsonpath="{.status.loadBalancer.ingress[0].hostname}")
until nslookup $API_ELB_PUBLIC_FQDN >/dev/null 2>&1; do sleep 2 && echo waiting for DNS to propagate...; done
curl $API_ELB_PUBLIC_FQDN/ok
echo
}
```

Test API endpoints:
```bash
curl -s $API_ELB_PUBLIC_FQDN/languages | jq .
curl -s $API_ELB_PUBLIC_FQDN/languages/go | jq .
curl -s $API_ELB_PUBLIC_FQDN/languages/java | jq .
```

> **Why?** The API is the backend that handles vote requests. The LoadBalancer gives it a public IP so the frontend can reach it from anywhere.

---

### Step 16: Deploy the Frontend

```bash
kubectl apply -f frontend-deployment.yaml
```

Expose frontend via LoadBalancer:
```bash
kubectl expose deploy frontend \
  --name=frontend \
  --type=LoadBalancer \
  --port=80 \
  --target-port=8080
```

Get the API's external IP and update the frontend config:
```bash
{
API_ELB_PUBLIC_FQDN=$(kubectl get svc api -ojsonpath="{.status.loadBalancer.ingress[0].hostname}")
echo API_ELB_PUBLIC_FQDN=$API_ELB_PUBLIC_FQDN
}
```

Edit the frontend deployment to add the API URL:
```bash
nano frontend-deployment.yaml
# Paste the API external IP in the appropriate env variable value
```

Restart the frontend to apply changes:
```bash
kubectl rollout restart deployment frontend -n cloudchamp
```

> **Why?** The frontend needs to know the API's address to send vote requests. After updating the config, a rollout restart forces new pods to pick up the latest settings.

---

### Step 17: Access the Application

Wait for the Frontend ELB to be ready:
```bash
{
FRONTEND_ELB_PUBLIC_FQDN=$(kubectl get svc frontend -ojsonpath="{.status.loadBalancer.ingress[0].hostname}")
until nslookup $FRONTEND_ELB_PUBLIC_FQDN >/dev/null 2>&1; do sleep 2 && echo waiting for DNS to propagate...; done
curl -I $FRONTEND_ELB_PUBLIC_FQDN
}
```

Get the app URL:
```bash
echo http://$FRONTEND_ELB_PUBLIC_FQDN
```

Open the URL in your browser, cast your votes, then verify votes are stored in MongoDB:
```bash
kubectl exec -it mongo-0 -- mongo langdb --eval "db.languages.find().pretty()"
```

> **Why?** This confirms the full end-to-end flow is working — votes from the browser are reaching the API and being stored in MongoDB.

---

## 🗺️ Architecture Overview

```
Browser
   |
   ▼
Frontend Service (LoadBalancer)
   |
   ▼
Frontend Pods (React)
   |
   ▼
API Service (LoadBalancer)
   |
   ▼
API Pods (Go)
   |
   ▼
MongoDB StatefulSet (mongo-0 PRIMARY, mongo-1, mongo-2 SECONDARY)
   |
   ▼
PersistentVolumes (EBS)
```

---

## 🎓 What You Learn from This Project

1. **Containerization** — Packaging apps with Docker
2. **Kubernetes Orchestration** — Managing containers at scale with EKS
3. **Microservices Architecture** — Decoupled frontend and backend
4. **Database Replication** — MongoDB Replica Set for high availability
5. **Secrets Management** — Securing credentials with Kubernetes Secrets
6. **Stateful Applications** — Using StatefulSets for databases
7. **Persistent Storage** — PersistentVolumes for data durability

---

## 📺 Reference

- YouTube Tutorial: [Watch here](https://youtu.be/pTmIoKUeU-A)
- Channel: [@cloudchamp](https://www.youtube.com/@cloudchamp?sub_confirmation=1)

---

## 📝 Summary

This project deploys a cloud-native voting app on AWS EKS with a MongoDB ReplicaSet backend, a Go API, and a React frontend — all managed by Kubernetes. Once live, votes cast in the browser are stored in MongoDB, demonstrating a complete end-to-end cloud-native application.
