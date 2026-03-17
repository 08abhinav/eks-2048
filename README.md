# AWS EKS Infrastructure with 2048 Demo Application

This project demonstrates end-to-end deployment of a sample 2048 game application on Amazon Elastic Kubernetes Service using `eksctl`, Kubernetes manifests, AWS Fargate profiles, Ingress, and the AWS Load Balancer Controller for Application Load Balancer integration.

It is designed as a practical hands-on project to understand how managed Kubernetes infrastructure works in AWS, including cluster provisioning, pod scheduling, service exposure, and ingress-based traffic routing.

---

## Architecture Overview

* Create EKS cluster using `eksctl`
* Configure AWS Fargate profile for serverless pod execution
* Deploy application using Kubernetes Deployment and Service
* Expose application through Kubernetes Ingress
* Install AWS Load Balancer Controller
* Route external traffic using Application Load Balancer (ALB)

---

## Tech Stack

* AWS EKS
* eksctl CLI
* Kubernetes
* AWS Fargate
* AWS IAM
* AWS ALB
* kubectl

---

## Project Workflow

### 1. Create EKS Cluster

Provision the Kubernetes cluster using `eksctl` with managed configuration.

```bash
eksctl create cluster --name game-cluster --region us-east-1 --fargate
```

## Note:

You can also create the cluster manually, but using `eksctl` is faster and less complicated for learning and quick setup.

---

### 2. Create Fargate Profile

A Fargate profile in AWS EKS defines which Kubernetes pods should run on AWS Fargate serverless infrastructure instead of EC2 worker nodes.

```bash
eksctl create fargateprofile \
    --cluster game-cluster \
    --region us-east-1 \
    --name alb-sample-app \
    --namespace game-2048
```

Here we are creating a new Fargate profile so that pods inside the `game-2048` namespace run on Fargate. By default, a Fargate profile named `fp-default` is created, which manages only the `default` and `kube-system` namespaces.

---

### 3. Update kubeconfig

```bash
aws eks update-kubeconfig --name game-cluster --region us-east-1
```

This command updates the local kubeconfig file so `kubectl` can connect to the EKS cluster.

---

### 4. Deploy Deployment, Service, and Ingress

You can apply the manifest directly using:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
```

But I prefer using:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml --dry-run=client -o yaml > 2048-manifest.yml
```

This creates a local manifest file containing the Deployment, Service, and Ingress configuration.

That way, you can review the configuration before applying it:

```bash
kubectl apply -f 2048-manifest.yml
```

Check all created resources using:

```bash
kubectl get all -n game-2048
```

### Note:

Check the ingress resource:

```bash
kubectl get ingress -n game-2048
```

You will notice that the address field is blank initially because the load balancer has not been created yet. It will be updated automatically with the load balancer URL after the ALB Controller is installed in Step 7.

---

### 5. Configure OIDC (OpenID Connect)

OIDC is required so Kubernetes service accounts can securely assume AWS IAM roles.

```bash
eksctl utils associate-iam-oidc-provider --cluster game-cluster --approve
```

We need the IAM OIDC provider because the ALB Controller requires permission to interact with AWS resources such as the Application Load Balancer.

In the next step, we will create a Kubernetes resource called **AWS Load Balancer Controller**. It runs as a Kubernetes pod, and since this pod needs to communicate with AWS services, it must use an IAM role.

OIDC makes this possible by connecting Kubernetes service accounts with AWS IAM roles, allowing secure communication without hardcoding credentials.

---

### 6. Create IAM Policy and Attach It to an IAM Role

Download IAM policy:

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json
```

Create IAM policy:

```bash
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

Create IAM role:

```bash
eksctl create iamserviceaccount \
  --cluster=game-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

---

### 7. Deploy ALB Controller

Using this Helm chart, the controller will be created, and it will use the `iamserviceaccount` for running the pod.

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks
```

Install the Helm chart:

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system \
  --set clusterName=game-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=<your-vpc-id>
```

Verify that the deployment is running:

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

Check all created resources in the `kube-system` namespace:

```bash
kubectl get all -n kube-system
```

What happens here is:

We create the **aws-load-balancer-controller**, which acts as an **Ingress Controller**. This controller watches the Ingress resource and automatically creates the AWS Load Balancer based on that configuration.

---

## Kubernetes Resources Used

* Deployment
* Service
* Ingress
* Namespace
* Fargate Profile