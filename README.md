# EKS Fargate Deployment with AWS Load Balancer Controller

This guide provides a comprehensive walkthrough for deploying a sample web application to a serverless Amazon EKS cluster using AWS Fargate and an AWS Load Balancer.

---

## 📘 Conceptual Overview

This project demonstrates a modern cloud deployment strategy using a combination of key AWS and Kubernetes technologies to create a scalable, highly available, and easily manageable application infrastructure.

- **Amazon EKS (Elastic Kubernetes Service):** Managed Kubernetes service that abstracts control plane management, offering scalability, security, and high availability out of the box.
- **AWS Fargate:** Serverless compute engine for containers, eliminating the need to manage EC2 instances for Kubernetes worker nodes.
- **AWS Load Balancer Controller:** Automatically provisions and configures Application Load Balancers (ALBs) in response to Kubernetes Ingress resources.
- **OIDC Identity Provider:** Enables Kubernetes service accounts to securely assume AWS IAM roles using short-lived credentials.

---
## 🏗️ **High-Level Architecture Overview**

```
                          ┌────────────────────────────────────┐
                          │        End User (Browser)         │
                          └────────────────────────────────────┘
                                       │
                                       ▼
                        ┌─────────────────────────────┐
                        │  Application Load Balancer  │
                        └─────────────────────────────┘
                                       │
                           (Ingress managed by ALB Controller)
                                       │
                                       ▼
                      ┌────────────────────────────────────┐
                      │        EKS Control Plane (AWS)     │
                      └────────────────────────────────────┘
                                       │
                                       ▼
            ┌──────────────────────────────────────────────────────┐
            │             Fargate (Serverless Compute)             │
            │  ┌──────────────────────────┐  ┌──────────────────┐  │
            │  │ Pod: 2048 Game App       │  │ Pod: Other Pods  │  │
            │  └──────────────────────────┘  └──────────────────┘  │
            └──────────────────────────────────────────────────────┘
                                       │
                                       ▼
        ┌────────────────────────────────────────────────────┐
        │ Kubernetes Namespace: `game-2048`                  │
        └────────────────────────────────────────────────────┘

        IAM & OIDC allow pods to securely access AWS services.
```

---

## 🔧 Components

* **End User:** Accesses the app via a public URL.
* **ALB (Application Load Balancer):** Created by the AWS Load Balancer Controller in response to an Ingress resource.
* **Ingress Resource:** Defined in Kubernetes to expose the 2048 game service externally.
* **AWS Load Balancer Controller:** Monitors Kubernetes Ingresses and Services, and creates ALBs.
* **EKS (Amazon Elastic Kubernetes Service):** Manages the Kubernetes control plane.
* **AWS Fargate:** Runs the containerized workloads without EC2 nodes.
* **IAM OIDC Provider:** Enables Kubernetes service accounts to assume IAM roles.

---

## ✅ Prerequisites

Make sure you have the following tools installed and configured:

- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [eksctl](https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html)
- [Helm](https://helm.sh/docs/intro/install/)

> Configure AWS CLI using:
```bash
aws configure
````

---

## 🚀 Step-by-Step Deployment

### 1. Create the EKS Cluster with Fargate

```bash
eksctl create cluster \
  --name demo-cluster \
  --region us-east-1 \
  --fargate
```

> 📌 This process typically takes 15-20 minutes.

---

### 2. Update kubeconfig

```bash
aws eks update-kubeconfig \
  --name demo-cluster \
  --region ap-south-1
```

> ✅ Ensures `kubectl` communicates with your new cluster.

---

### 3. Create a Fargate Profile

```bash
eksctl create fargateprofile \
  --cluster demo-cluster \
  --region ap-south-1 \
  --name alb-sample-app \
  --namespace game-2048
```

---

### 4. Deploy the 2048 Game Application

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
```

> ✅ Verify resources:

```bash
kubectl get pods -n game-2048
kubectl get svc -n game-2048
kubectl get ingress -n game-2048
```

---

### 5. Attach the OIDC Provider

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster demo-cluster \
  --approve
```

---

### 6. Create IAM Policy and Role for Load Balancer Controller

#### a. Download IAM Policy

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json
```

#### b. Create IAM Policy

```bash
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json
```

#### c. Create IAM Role and Kubernetes Service Account

Replace placeholders:

* `<your-cluster-name>`
* `<your_account_id>`

```bash
eksctl create iamserviceaccount \
  --cluster=<your-cluster-name> \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<your_account_id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

---

### 7. Install AWS Load Balancer Controller via Helm

Replace placeholders:

* `<your-cluster-name>`
* `<your-region>`
* `<your-vpc-id>`

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=<your-cluster-name> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=<your-region> \
  --set vpcId=<your-vpc-id>
```

> ✅ Confirm controller deployment:

```bash
kubectl get deployment aws-load-balancer-controller -n kube-system
```

---

### 8. Access the Application

Get the Ingress hostname:

```bash
kubectl get ingress -n game-2048
```

Visit the URL shown under `ADDRESS`, for example:

```
k8s-game2048-ingress2-bcac0b5b37-1186747725.ap-south-1.elb.amazonaws.com
```

Paste it into your browser to access the 2048 game.

---

## 🧹 Cleanup

To avoid incurring costs:

```bash
eksctl delete cluster --name demo-cluster --region us-east-1
```

This command deletes the EKS cluster and all associated resources including:

* Fargate profiles
* Load balancers
* IAM roles
* Application workloads

---

## 📂 Resources

* [AWS EKS Documentation](https://docs.aws.amazon.com/eks/)
* [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)
* [Fargate on EKS](https://docs.aws.amazon.com/eks/latest/userguide/fargate.html)
* [Helm Charts](https://artifacthub.io/packages/search?repo=eks)

---
