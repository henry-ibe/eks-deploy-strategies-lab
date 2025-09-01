# Ship Safely on EKS: RollingUpdate, Recreate, Blue/Green & Canary (Hands‑On Lab)

**Objective.** Build a small, cost‑aware EKS cluster and demonstrate four deployment strategies—RollingUpdate, Recreate, Blue/Green, and Canary—validating each one with real traffic.

---

## Table of contents
1. Why deployment strategies matter (what you’ll learn)
2. Prerequisites
3. Architecture at a glance
4. Guardrails (budget & workspace)
5. Create an EKS cluster (cost‑friendly)
6. Public access via Service: LoadBalancer
7. Base app & Service
8. Strategy 1: RollingUpdate (native K8s)
9. Strategy 2: Recreate (native K8s)
10. Install Argo Rollouts (progressive delivery)
11. Strategy 3: Blue/Green (Argo Rollouts, 2 LoadBalancers)
12. Strategy 4: Canary (Argo Rollouts, replica‑weighted)
13. Observability & verification (CLI + optional dashboard)
14. Cleanup & cost check

---

## 1) Why deployment strategies matter
**What.** Strategies let you ship new versions with the right balance of **safety, speed, and cost**.

**Why.** In production you need to reduce risk (no/low downtime, fast rollback) while moving quickly. Platform/DevOps/SRE folks are expected to know these patterns cold.

You’ll practice:
- **RollingUpdate** – Gradually replace pods; minimal/no downtime.
- **Recreate** – Stop old first, then start new; simple; brief outage.
- **Blue/Green** – Two environments; instant cutover & rollback; safe preview.
- **Canary** – Gradual promotion with pauses (replica‑weighted in this lab).

---

## 2) Prerequisites
- An **AWS account** with permission to create EKS, VPC, EC2, ELB, IAM (Admin for labs is fine).
- A small **EC2 Linux instance** to act as your workstation (this guide uses Amazon Linux 2 on `t3.micro` or `t3.small`).
- Basic terminal comfort.
- Optional: a GitHub repo to store the manifests.

> **Cost note.** EKS control plane is billed per hour; Load Balancers also cost money. We keep everything tiny and show full cleanup steps.

---

## 3) Architecture at a glance
- 1× **EKS cluster** in `us-east-1` with **one spot node**.
- Public **Service: LoadBalancer** for direct access (no Ingress required).
- Argo Rollouts controller for Blue/Green and Canary.

---

## 4) Guardrails (budget & workspace)
**What.** Create a small monthly budget and a notification in AWS Billing.

**Why.** Labs can leak cost; alerts prevent surprises.

> Tip: Tag lab resources with `project=eks-deploy-lab` so they’re easy to find and clean.

---

## 5) Create an EKS workstation on EC2
**Launch instance**
- **Name:** `eks-workstation`
- **AMI:** Amazon Linux 2 (x86_64)
- **Type:** `t3.micro` (ok) or `t3.small` (more pods)
- **Key pair:** *Proceed without* if you’ll use **EC2 Instance Connect**
- **Network:** Default VPC, public subnet, **Auto‑assign public IP: Enabled**
- **Security group:** allow SSH 22 from your IP
- **IAM role:** attach a role with **AdministratorAccess** (simplest for labs)
- **Instance metadata (IMDS):** enabled

**Install tools**
```bash
# base updates
sudo yum -y update

# kubectl (stable)
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

# eksctl
curl -sSL "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz" -o eksctl.tgz
tar xzf eksctl.tgz && rm eksctl.tgz
sudo mv eksctl /usr/local/bin/

# helm (optional for this lab)
curl -sSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# argo rollouts kubectl plugin (CLI)
curl -sSL -o kubectl-argo-rollouts   https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
chmod +x kubectl-argo-rollouts && sudo mv kubectl-argo-rollouts /usr/local/bin/

# quick check
kubectl version --client
eksctl version
helm version
kubectl argo rollouts version
```

---

## 6) Create an EKS cluster (cost‑friendly)
**What.** Tiny EKS with a single **spot** node.

**Why.** Managed control plane + spot worker keeps costs low but realistic.

```bash
eksctl create cluster -f k8s/cluster.yaml
```

**Verify**
```bash
eksctl get cluster --region us-east-1
kubectl get nodes
kubectl get pods -A
```
> **t3.micro note.** It supports very few pods. If you see `Too many pods` during the lab, scale the nodegroup to 2 nodes:
```bash
eksctl scale nodegroup --cluster k8s-deploy-project --region us-east-1 --name ng-spot --nodes 2
```

---

## 7) Public access via LoadBalancer
Create a namespace and a public Service the app will use across all strategies.

```bash
kubectl create namespace app
kubectl apply -f k8s/svc-stable.yaml

# wait for DNS
kubectl -n app get svc rollout-svc -w
LB=$(kubectl -n app get svc rollout-svc -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "LoadBalancer: http://$LB"
```

---

## 8) Strategy 1 — RollingUpdate (native K8s)
```bash
kubectl apply -f k8s/deploy-rolling.yaml
kubectl -n app rollout status deploy/demo-rolling

# upgrade to V2
kubectl -n app set image deploy/demo-rolling web=public.ecr.aws/nginx/nginx:1.27-alpine
kubectl -n app rollout status deploy/demo-rolling

# rollback
kubectl -n app rollout undo deploy/demo-rolling
```

---

## 9) Strategy 2 — Recreate (native K8s)
```bash
kubectl -n app scale deploy/demo-rolling --replicas=0
kubectl apply -f k8s/deploy-recreate.yaml
kubectl -n app rollout status deploy/demo-recreate

# upgrade (expect a brief gap)
kubectl -n app set image deploy/demo-recreate web=public.ecr.aws/nginx/nginx:1.27-alpine
kubectl -n app rollout status deploy/demo-recreate
```

---

## 10) Install Argo Rollouts
```bash
kubectl create namespace argo-rollouts || true
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
kubectl -n argo-rollouts wait deploy/argo-rollouts --for=condition=Available --timeout=120s
kubectl -n argo-rollouts get pods
```

---

## 11) Strategy 3 — Blue/Green (Argo Rollouts, 2 LBs)

```bash
kubectl apply -f k8s/svc-preview.yaml
kubectl apply -f k8s/rollout-bluegreen.yaml

# propose new version (preview only)
kubectl argo rollouts set image demo-bglab web=public.ecr.aws/nginx/nginx:1.27-alpine -n app

kubectl argo rollouts get rollout demo-bglab -n app -w

# endpoints
LB_STABLE=$(kubectl -n app get svc rollout-svc -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
LB_PREVIEW=$(kubectl -n app get svc rollout-svc-preview -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "Stable:  http://$LB_STABLE"
echo "Preview: http://$LB_PREVIEW"

# promote when happy
kubectl argo rollouts promote demo-bglab -n app

# rollback
kubectl argo rollouts undo demo-bglab -n app
```

---

## 12) Strategy 4 — Canary (replica‑weighted)

> This lab uses replica weighting (50%/100%) without a traffic router. For precise % routing, add NGINX Ingress, ALB, or a service mesh.

```bash
kubectl apply -f k8s/rollout-canary.yaml

kubectl argo rollouts set image demo-canary web=public.ecr.aws/nginx/nginx:1.27-alpine -n app
kubectl argo rollouts get rollout demo-canary -n app -w

kubectl argo rollouts promote demo-canary -n app --full
# rollback: kubectl argo rollouts undo demo-canary -n app
```

---

## 13) Observability & verification

**Live watcher (script)**

```bash
./scripts/watch-rollout.sh
```

**Quick HTTP checks**
```bash
LB_S=$(kubectl -n app get svc rollout-svc -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
LB_P=$(kubectl -n app get svc rollout-svc-preview -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
curl -I "http://$LB_S" | head -n1
curl -I "http://$LB_P" | head -n1
```

**Optional web UI (run on your laptop)**
```bash
aws eks update-kubeconfig --region us-east-1 --name k8s-deploy-project
kubectl argo rollouts dashboard -n app   # http://localhost:3100/rollouts
```

---

## 14) Cleanup & costs
```bash
kubectl delete ns app argo-rollouts --ignore-not-found
eksctl delete cluster -f k8s/cluster.yaml
```

**Optional AWS cleanup checks:** verify no leftover CloudFormation stacks, load balancers, target groups, SGs, ENIs, or unattached EBS volumes.

---

## License
MIT © 2025 Henry Ibe
