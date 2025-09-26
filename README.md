# **EKS Fargate Deployment with AWS Load Balancer Controller**

This guide provides a comprehensive walkthrough for deploying a sample web application to a serverless Amazon EKS cluster using AWS Fargate and an AWS Load Balancer. The steps are based on the project explained in the YouTube video: [https://youtu.be/RRCrY12VY\_s](https://www.google.com/search?q=https://youtu.be/RRCrY12VY_s).

## **Conceptual Overview**

This project demonstrates a modern cloud deployment strategy using a combination of key AWS and Kubernetes technologies to create a scalable, highly available, and easily manageable application infrastructure.

* **Amazon EKS (Elastic Kubernetes Service):** As a managed service for running Kubernetes, EKS abstracts away the complexity of managing the control plane. This means AWS handles the patching, scaling, and high availability of the Kubernetes master nodes, allowing developers and DevOps teams to focus on their applications rather than the underlying infrastructure. It provides a robust and secure foundation for containerized workloads.  
* **AWS Fargate:** This serverless compute engine for containers works seamlessly with Amazon EKS. With Fargate, you don't need to provision, configure, or scale EC2 instances for your worker nodes. Instead, you simply define your pods, and Fargate automatically provisions the right amount of compute capacity to run your containers. This pay-as-you-go model for compute resources offers significant cost savings and operational simplicity, as you are billed only for the resources your containers consume.  
* **AWS Load Balancer Controller:** This is a Kubernetes controller that directly manages AWS Elastic Load Balancers for a Kubernetes cluster. When you define a Kubernetes Ingress resource, this controller automatically provisions and configures a corresponding Application Load Balancer (ALB) in your AWS account. It intelligently routes external traffic from the internet to the correct services and pods within your EKS cluster, handling TLS termination and advanced routing rules, and ensuring high availability.  
* **OIDC Identity Provider:** An OpenID Connect (OIDC) identity provider for your EKS cluster enables secure authentication between Kubernetes service accounts and AWS IAM. This allows pods running in the cluster to assume IAM roles and interact with other AWS services (like S3, DynamoDB, or the Load Balancer Controller itself) using temporary, scoped credentials without storing long-lived AWS keys within the pod. This practice significantly enhances the security of your cloud environment.

## **Prerequisites**

Before starting this project, ensure you have the following command-line tools installed and configured on your system. The latest versions are recommended for the best experience.

1. **AWS CLI:** The official command-line interface for managing AWS services.  
2. **kubectl:** The standard command-line tool for interacting with Kubernetes clusters.  
3. **eksctl:** A simple, high-level CLI for creating and managing EKS clusters.  
4. **Helm:** The package manager for Kubernetes, used to deploy complex applications.

After installation, configure your AWS CLI with your credentials. This step is essential to grant the CLIs permission to create and manage resources in your AWS account.

aws configure

## **Step-by-Step Deployment**

Follow these steps in order to deploy the 2048 game application to your EKS cluster, leveraging Fargate and the AWS Load Balancer Controller.

### **1\. Create the EKS Cluster with Fargate**

Use the eksctl command to create a new cluster. This command will provision a cluster named demo-cluster in the us-east-1 region with Fargate enabled. This process sets up the control plane, networking, and all necessary components, and it can take a significant amount of time (typically 15-20 minutes).

eksctl create cluster \--name demo-cluster \--region us-east-1 \--fargate

### **2\. Update kubeconfig**

After the cluster is created, it's crucial to update your local kubeconfig file. This command configures kubectl to point to your new EKS cluster, allowing you to run kubectl commands against it.

aws eks update-kubeconfig \--name demo-cluster \--region ap-south-1

### **3\. Create a Fargate Profile**

Create a Fargate profile for the application's namespace (game-2048). This profile tells EKS that any pods deployed to this namespace should be scheduled on Fargate instead of on a self-managed worker node. This is a key step in enabling the serverless functionality of Fargate.

eksctl create fargateprofile \\  
    \--cluster demo-cluster \\  
    \--region ap-south-1 \\  
    \--name alb-sample-app \\  
    \--namespace game-2048

### **4\. Deploy the 2048 Game Application**

Apply the Kubernetes manifest file for the 2048 game directly from its GitHub repository. This single command creates the Deployment and Service objects needed to run the application in your cluster.

kubectl apply \-f \[https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048\_full.yaml\](https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048\_full.yaml)

After applying the manifest, verify that the pods, services, and ingress are created successfully using these kubectl commands.

kubectl get pods \-n game-2048   
kubectl get svc \-n game-2048  
kubectl get ingress \-n game-2048

### **5\. Attach the OIDC Provider**

This step associates an IAM OIDC provider with your cluster. This identity provider allows Kubernetes service accounts to assume IAM roles, which is a key security best practice for granting permissions to applications.

eksctl utils associate-iam-oidc-provider \--cluster demo-cluster \--approve

### **6\. Create IAM Policy and Role for the Load Balancer Controller**

The AWS Load Balancer Controller requires specific IAM permissions to create and manage ALBs in your AWS account. You must create an IAM policy and an IAM role with the necessary permissions.

First, download the official IAM policy JSON file:

curl \-O \[https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam\_policy.json\](https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam\_policy.json)

Next, create the IAM policy in your AWS account using the downloaded file:

aws iam create-policy \\  
    \--policy-name AWSLoadBalancerControllerIAMPolicy \\  
    \--policy-document file://iam\_policy.json

Finally, create a Kubernetes service account and an IAM role for the controller, and attach the policy to the role. **Remember to replace \<your-cluster-name\> and \<your\_account\_id\> with your actual values.**

eksctl create iamserviceaccount \\  
  \--cluster=\<your-cluster-name\> \\  
  \--namespace=kube-system \\  
  \--name=aws-load-balancer-controller \\  
  \--role-name AmazonEKSLoadBalancerControllerRole \\  
  \--attach-policy-arn=arn:aws:iam::\<your\_account\_id\>:policy/AWSLoadBalancerControllerIAMPolicy \\  
  \--approve

### **7\. Install the AWS Load Balancer Controller**

Use Helm, the Kubernetes package manager, to install the AWS Load Balancer Controller. This will deploy the controller into the kube-system namespace. Once installed, it will automatically detect the Ingress resource created in step 4 and provision an ALB.

helm repo add eks \[https://aws.github.io/eks-charts\](https://aws.github.io/eks-charts)  
helm repo update eks  
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \-n kube-system \\  
    \--set clusterName=\<your-cluster-name\> \\  
    \--set serviceAccount.create=false \\  
    \--set serviceAccount.name=aws-load-balancer-controller \\  
    \--set region=\<your-region\> \\  
    \--set vpcId=\<your-vpc-id\>

After installation, verify that the deployment is successful. You should see a READY status of 1/1 or 2/2 for the aws-load-balancer-controller deployment.

### **8\. Access the Application**

Once the controller has finished provisioning the ALB, the Ingress resource will be updated with a public hostname. Get the hostname to access your application.

kubectl get ingress \-n game-2048

Copy the address from the output (e.g., k8s-game2048-ingress2-bcac0b5b37-1186747725.ap-south-1.elb.amazonaws.com) and paste it into your browser to play the game. You can also verify that the load balancer was created in the AWS console under EC2 \> Load Balancers.

## **Cleanup**

To avoid incurring unexpected costs, it is critical to clean up all the resources you have created in this project. The simplest way to do this is to use the eksctl delete cluster command. This command will delete the EKS cluster and all associated resources, including the Fargate profiles, load balancers, and IAM roles created during the deployment process.

eksctl delete cluster \--name demo-cluster \--region us-east-1

