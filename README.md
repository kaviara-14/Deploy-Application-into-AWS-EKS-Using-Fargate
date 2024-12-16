# Deploy an Application on AWS EKS with Fargate and ALB Controller

This project provides a detailed walkthrough for deploying a Kubernetes application into AWS Elastic Kubernetes Service (EKS) using Fargate. It also covers configuring the AWS Load Balancer Controller (ALB Controller) to manage load balancers for your Kubernetes services.

---

## Prerequisites
- AWS CLI installed and configured
- `eksctl` installed
- `kubectl` installed
- `helm` installed
- Proper IAM permissions to manage EKS and associated AWS resources

---

## 1. Create a Cluster using Fargate

Fargate is a serverless compute engine for containers that eliminates the need to provision and manage servers. In this step, we create an EKS cluster with Fargate enabled to manage Kubernetes workloads without managing the underlying infrastructure.

```bash
eksctl create cluster --name demo-cluster --region us-east-1 --fargate
```

---

## 2. Create a Fargate Profile

Fargate profiles define which pods run on Fargate. Here, we specify the namespace for the pods that should use Fargate for compute.

```bash
eksctl create fargateprofile --cluster demo-cluster --region us-east-1 --name alb-sample-app --namespace game-2048
```

---

## 3. Deploy the Application

Deploying the application involves creating Kubernetes resources like Deployments, Services, and Ingress. This step uses a pre-configured YAML file to deploy a sample 2048 game application into the `game-2048` namespace.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
```

---

## 4. Configure IAM for AWS Load Balancer Controller

The AWS Load Balancer Controller manages AWS Elastic Load Balancers for Kubernetes clusters. Proper IAM configuration is essential for enabling the controller to interact with AWS services securely.

### - Associate IAM OIDC Provider

This step associates an OIDC identity provider with your EKS cluster, enabling IAM roles for Kubernetes service accounts.

```bash
eksctl utils associate-iam-oidc-provider --cluster $cluster_name --approve
```

### - Download IAM Policy

The IAM policy defines the permissions required for the AWS Load Balancer Controller to manage resources.

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json
```

### - Create IAM Policy

Create a custom IAM policy using the downloaded file to grant the necessary permissions.

```bash
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

### - Create IAM Role

Create a service account in Kubernetes and associate it with the custom IAM policy, enabling the AWS Load Balancer Controller to operate securely.

```bash
eksctl create iamserviceaccount \
  --cluster=<your-cluster-name> \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --region=<your-region> \
  --approve
```

---

## 5. Deploy the AWS Load Balancer Controller

The AWS Load Balancer Controller is installed using Helm, a package manager for Kubernetes. This controller automatically provisions and manages AWS load balancers for Kubernetes Services.

### - Add Helm Repository

Add the EKS Helm chart repository to your local Helm configuration.

```bash
helm repo add eks https://aws.github.io/eks-charts
```

### - Update the Helm Repository

Ensure you have the latest version of the EKS charts.

```bash
helm repo update
```

### - Install AWS Load Balancer Controller

Install the AWS Load Balancer Controller into the Kubernetes cluster. Here, we specify the cluster name, service account, region, and VPC ID.

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \            
  -n kube-system \
  --set clusterName=<your-cluster-name> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=<region> \
  --set vpcId=<your-vpc-id>
```

### - Verify Deployment

Ensure the AWS Load Balancer Controller is running successfully.

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

---

## Additional Notes
- Ensure all placeholders (e.g., `<your-cluster-name>`, `<region>`, `<your-vpc-id>`) are replaced with your actual values.
- Check AWS documentation for the latest updates on EKS and AWS Load Balancer Controller.
- Monitor the deployed application using `kubectl` commands.

---
