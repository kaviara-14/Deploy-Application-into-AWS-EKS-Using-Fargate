# Deploying a 2048 Game on AWS EKS Fargate with Load Balancing

This project provides a detailed walkthrough for deploying a Kubernetes application into AWS Elastic Kubernetes Service (EKS) using Fargate. It also covers configuring the AWS Load Balancer Controller (ALB Controller) to manage load balancers for your Kubernetes services.

![image](https://github.com/user-attachments/assets/fd35a0ba-3246-441b-8e52-48330327fa90)

---

## Steps for deployment

### 1. Create EKS Cluster

```bash
# This command creates an EKS cluster named “demo-cluster” in the us-east-1 region using Fargate.
 --name demo-cluster --region us-east-1 --fargate

# Now Update your local kubeconfig file to connect with kubectl .
aws eks --region us-east-1 update-kubeconfig --name demo-cluster

```
---

### 2. Create Fargate Profile and Namespace

This command prepares your cluster to run the application in a serverless Fargate environment and creates a dedicated namespace (game-2048) for it.

```bash
eksctl create fargateprofile --cluster demo-cluster --region us-east-1 --name alb-sample-app --namespace game-2048
```

---

### 3. . Deploy the 2048 Game Application:

In this URL are the Deployment, Service, and Ingress resources for the 2048 game application that was already created previously.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
```
---

## 4. Configure IAM OIDC Provider and Policy

The AWS Load Balancer Controller manages AWS Elastic Load Balancers for Kubernetes clusters. Proper IAM configuration is essential for enabling the controller to interact with AWS services securely.

```bash
# Now Associate an IAM OIDC provider with your cluster.
eksctl utils associate-iam-oidc-provider --cluster $cluster_name --approve

# Then Create an IAM policy for the AWS Load Balancer Controller.
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json.
aws iam create-policy \
        --policy-name AWSLoadBalancerControllerIAMPolicy \
        --policy-document file://iam_policy.json


# Now we creating the role and attaching the role to the ServiceAccount of the pod.This ensures that the pod has the necessary credentials (through the ServiceAccount) to integrate with other AWS resources.
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

```bash
#Install Application Load Balancer (ALB) Ingress controller through helm chart 
sudo snap install helm --classic
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n <name-space> --set clusterName=my-cluster --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller

# Now final check you have to do is you have to verify this load balancer is created and there are least two replicas of it     
kubectl get deployment -n kube-system aws-load-balancer-controller

```

---
