# 2048GameApp
# 🎮 2048 Game on Amazon EKS with Fargate

This project demonstrates how to deploy the **2048 web application** on **Amazon EKS using AWS Fargate** and expose it to the internet through an **AWS Application Load Balancer (ALB)** using the **AWS Load Balancer Controller**.

---

## 🏗️ Architecture

```text
                         Internet
                            │
                            ▼
                ┌───────────────────────┐
                │   AWS Application     │
                │   Load Balancer (ALB) │
                └───────────┬───────────┘
                            │
                            ▼
                ┌───────────────────────┐
                │ Kubernetes Ingress    │
                │   ingress-2048        │
                └───────────┬───────────┘
                            │
                            ▼
                ┌───────────────────────┐
                │ Kubernetes Service    │
                │    service-2048       │
                │      NodePort         │
                └───────────┬───────────┘
                            │
                            ▼
                ┌───────────────────────┐
                │    AWS Fargate        │
                │                       │
                │   2048 Application    │
                │      Pods             │
                └───────────────────────┘
```

### Request flow

```text
Browser
   ↓
Internet-facing ALB
   ↓
Kubernetes Ingress
   ↓
AWS Load Balancer Controller
   ↓
Target Group
   ↓
Fargate Pod IP
   ↓
2048 Application
```

---

# 📋 Prerequisites

The following environment was used for this project:

* Ubuntu 26.04 LTS
* AWS Account
* IAM credentials or IAM role with sufficient permissions
* AWS CLI
* kubectl
* eksctl
* Helm
* Amazon EKS
* AWS Fargate

Verify the installed tools:

```bash
aws --version
kubectl version --client
eksctl version
helm version
```

---

# ☁️ 1. Configure AWS CLI

Configure AWS credentials:

```bash
aws configure
```

Provide:

```text
AWS Access Key ID:     <YOUR_ACCESS_KEY>
AWS Secret Access Key: <YOUR_SECRET_KEY>
Default region:        us-east-1
Default output:        json
```

Verify AWS authentication:

```bash
aws sts get-caller-identity
```

The command should return your AWS account and identity information.

> **Security:** Never commit AWS access keys or secret keys to GitHub.

---

# ☸️ 2. Create the EKS Cluster

Create the EKS cluster using Fargate:

```bash
eksctl create cluster \
  --name 2048cluster \
  --region us-east-1 \
  --fargate
```

This creates the EKS cluster and the default Fargate profile.

Verify:

```bash
eksctl get cluster --region us-east-1
```

Check Kubernetes resources:

```bash
kubectl get pods -A
```

---

# 🚀 3. Create the Application Fargate Profile

Create a dedicated Fargate profile for the `game-2048` namespace:

```bash
eksctl create fargateprofile \
  --cluster 2048cluster \
  --region us-east-1 \
  --name alb-sample-app \
  --namespace game-2048
```

Verify:

```bash
aws eks list-fargate-profiles \
  --cluster-name 2048cluster \
  --region us-east-1
```

Expected profiles include:

```text
fp-default
alb-sample-app
```

---

# 🔐 4. Associate IAM OIDC Provider

The AWS Load Balancer Controller uses IAM roles for service accounts.

Associate the OIDC provider:

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster 2048cluster \
  --region us-east-1 \
  --approve
```

---

# 🔑 5. Create IAM Policy for AWS Load Balancer Controller

Download the IAM policy:

```bash
curl -o iam-policy.json \
https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
```

Create the IAM policy:

```bash
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam-policy.json
```

The policy ARN will look like:

```text
arn:aws:iam::<AWS_ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy
```

---

# 👤 6. Create IAM Service Account

Create the Kubernetes service account and associate it with the IAM policy:

```bash
eksctl create iamserviceaccount \
  --cluster 2048cluster \
  --region us-east-1 \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --attach-policy-arn arn:aws:iam::<AWS_ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --approve
```

Verify:

```bash
kubectl get serviceaccount \
  aws-load-balancer-controller \
  -n kube-system
```

Also:

```bash
eksctl get iamserviceaccount \
  --cluster 2048cluster \
  --region us-east-1
```

---

# ⎈ 7. Install AWS Load Balancer Controller

Add the AWS EKS Helm repository:

```bash
helm repo add eks https://aws.github.io/eks-charts
```

Update the Helm repository:

```bash
helm repo update
```

Get the VPC ID:

```bash
aws eks describe-cluster \
  --name 2048cluster \
  --region us-east-1 \
  --query "cluster.resourcesVpcConfig.vpcId" \
  --output text
```

Install the AWS Load Balancer Controller:

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=2048cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=$(aws eks describe-cluster \
    --name 2048cluster \
    --region us-east-1 \
    --query "cluster.resourcesVpcConfig.vpcId" \
    --output text)
```

---

# ✅ 8. Verify AWS Load Balancer Controller

Check the controller pods:

```bash
kubectl get pods -n kube-system
```

Expected:

```text
aws-load-balancer-controller-xxxxx   1/1   Running
aws-load-balancer-controller-xxxxx   1/1   Running
```

Check the deployment:

```bash
kubectl get deployment \
  aws-load-balancer-controller \
  -n kube-system
```

Check the webhook:

```bash
kubectl get endpoints \
  aws-load-balancer-webhook-service \
  -n kube-system
```

The webhook should have pod endpoints.

---

# 🎮 9. Deploy the 2048 Application

Create the application namespace:

```bash
kubectl create namespace game-2048
```

Deploy the 2048 application:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.7.2/docs/examples/2048/2048_full.yaml
```

Verify the pods:

```bash
kubectl get pods -n game-2048
```

The pods should eventually show:

```text
Running
```

---

# 🔎 10. Verify the Service

Check the Service:

```bash
kubectl get svc -n game-2048
```

The application Service is expected to be a `NodePort` in this configuration.

Example:

```text
NAME          TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)
service-2048  NodePort   10.x.x.x        <none>        80:xxxxx/TCP
```

### Important

Do **not** change the Service to `LoadBalancer`.

The architecture uses:

```text
ALB
 ↓
Ingress
 ↓
NodePort Service
 ↓
Fargate Pods
```

The AWS Load Balancer Controller creates the ALB from the Kubernetes Ingress.

---

# 🌐 11. Configure the Ingress

The Ingress uses:

```yaml
alb.ingress.kubernetes.io/scheme: internet-facing
alb.ingress.kubernetes.io/target-type: ip
```

The important setting for Fargate is:

```yaml
alb.ingress.kubernetes.io/target-type: ip
```

This allows the ALB target group to use the **Fargate pod IP addresses**.

Check the Ingress:

```bash
kubectl get ingress -n game-2048
```

Expected output:

```text
NAME           CLASS   HOSTS   ADDRESS
ingress-2048   alb     *       k8s-xxxxx.us-east-1.elb.amazonaws.com
```

---

# 🌍 12. Access the 2048 Application

Get the ALB hostname:

```bash
kubectl get ingress ingress-2048 \
  -n game-2048
```

Copy the value under `ADDRESS`.

Open:

```text
http://<ALB-DNS-NAME>
```

Example:

```text
http://k8s-game2048-ingress2-xxxxx.us-east-1.elb.amazonaws.com
```

The 2048 game should now be accessible from your browser.

---

# 🔍 13. Useful Verification Commands

### Check all pods

```bash
kubectl get pods -A
```

### Check 2048 pods

```bash
kubectl get pods -n game-2048
```

### Check Service

```bash
kubectl get svc -n game-2048
```

### Check Service endpoints

```bash
kubectl get endpoints service-2048 -n game-2048
```

Or, for newer Kubernetes versions:

```bash
kubectl get endpointslice -n game-2048
```

### Check Ingress

```bash
kubectl get ingress -n game-2048
```

### Describe Ingress

```bash
kubectl describe ingress ingress-2048 -n game-2048
```

### Check Load Balancer Controller

```bash
kubectl get pods -n kube-system \
  -l app.kubernetes.io/name=aws-load-balancer-controller
```

### Controller logs

```bash
kubectl logs -n kube-system \
  deployment/aws-load-balancer-controller
```

---

# 🛠️ Troubleshooting

## ALB ADDRESS is empty

Check:

```bash
kubectl describe ingress ingress-2048 -n game-2048
```

Look at the `Events` section.

Also check:

```bash
kubectl logs -n kube-system \
  deployment/aws-load-balancer-controller \
  --tail=100
```

---

## AccessDenied: DescribeListenerAttributes

If you see:

```text
not authorized to perform:
elasticloadbalancing:DescribeListenerAttributes
```

Make sure the IAM policy contains:

```text
elasticloadbalancing:DescribeListenerAttributes
```

Check:

```bash
aws iam get-policy \
  --policy-arn arn:aws:iam::<AWS_ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy
```

Get the default policy version:

```bash
aws iam get-policy-version \
  --policy-arn arn:aws:iam::<AWS_ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --version-id <VERSION_ID>
```

If necessary, update the policy using the current AWS Load Balancer Controller policy:

```bash
curl -o iam-policy-new.json \
https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
```

Then:

```bash
aws iam create-policy-version \
  --policy-arn arn:aws:iam::<AWS_ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam-policy-new.json \
  --set-as-default
```

Restart the controller:

```bash
kubectl rollout restart deployment/aws-load-balancer-controller \
  -n kube-system
```

Verify:

```bash
kubectl rollout status deployment/aws-load-balancer-controller \
  -n kube-system
```

---

## Webhook has no endpoints

If you see:

```text
no endpoints available for service
"aws-load-balancer-webhook-service"
```

Check:

```bash
kubectl get pods -n kube-system \
  -l app.kubernetes.io/name=aws-load-balancer-controller
```

Then:

```bash
kubectl get endpoints \
  aws-load-balancer-webhook-service \
  -n kube-system
```

The service should have controller pod IPs as endpoints.

---

## Service has no endpoints

Check:

```bash
kubectl get endpoints service-2048 -n game-2048
```

If no endpoints are listed, check:

```bash
kubectl get pods -n game-2048 -o wide
```

Then inspect the Service:

```bash
kubectl describe svc service-2048 -n game-2048
```

The Service selector must match the labels on the 2048 pods.

---

# 🧹 Cleanup

When you no longer need the environment, delete the EKS cluster:

```bash
eksctl delete cluster \
  --name 2048cluster \
  --region us-east-1
```

This removes the EKS cluster and resources managed by `eksctl`.

You can verify:

```bash
aws eks list-clusters \
  --region us-east-1
```

---

# 🗑️ Optional IAM Cleanup

If you are completely finished with the project, you can also remove the IAM service account:

```bash
eksctl delete iamserviceaccount \
  --cluster 2048cluster \
  --region us-east-1 \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --approve
```

If you no longer need the IAM policy:

```bash
aws iam delete-policy \
  --policy-arn arn:aws:iam::<AWS_ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy
```

Only delete the policy after confirming it isn't being used by another application or IAM role.

---

# 📁 Project Structure

A recommended repository structure is:

```text
2048game-k8s/
│
├── .github/
│   └── workflows/
│       └── hello-world.yml
│
├── k8s/
│   ├── namespace.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ingress.yaml
│
├── README.md
└── .gitignore
```

---

# 🔄 Deployment Flow

```text
Developer
    │
    ▼
GitHub Repository
    │
    ▼
Kubernetes Manifests
    │
    ▼
Amazon EKS
    │
    ▼
AWS Fargate
    │
    ▼
2048 Pods
    │
    ▼
Kubernetes Service
    │
    ▼
AWS Load Balancer Controller
    │
    ▼
Application Load Balancer
    │
    ▼
Internet
```

---

# 🔐 Security Notes

* Never commit AWS access keys or secret keys.
* Never store AWS credentials in Kubernetes manifests.
* Use IAM roles for Kubernetes workloads where possible.
* Use the AWS Load Balancer Controller IAM service account instead of attaching broad permissions directly to worker nodes.
* Avoid using `AdministratorAccess` for production workloads.
* Restrict security group access according to your environment.
* Delete unused AWS resources to avoid unnecessary charges.

---

# 🧰 Technologies Used

| Technology                   | Purpose                            |
| ---------------------------- | ---------------------------------- |
| AWS EC2                      | Administration / deployment host   |
| AWS CLI                      | AWS resource management            |
| Amazon EKS                   | Kubernetes cluster                 |
| AWS Fargate                  | Serverless Kubernetes compute      |
| kubectl                      | Kubernetes CLI                     |
| eksctl                       | EKS cluster management             |
| Helm                         | Kubernetes package management      |
| AWS Load Balancer Controller | Creates/manages AWS ALB            |
| Application Load Balancer    | Public access to application       |
| Kubernetes Ingress           | Routes traffic to the 2048 Service |
| Kubernetes Service           | Exposes the application internally |

---

# 🎯 Final Result

The 2048 application is deployed on **Amazon EKS using AWS Fargate** and exposed publicly through an **internet-facing AWS Application Load Balancer**.

```text
                    🌎 Internet
                         │
                         ▼
              ┌─────────────────────┐
              │    AWS ALB          │
              │ Internet-facing      │
              └──────────┬──────────┘
                         │
                         ▼
              ┌─────────────────────┐
              │ Kubernetes Ingress  │
              │   ingress-2048      │
              └──────────┬──────────┘
                         │
                         ▼
              ┌─────────────────────┐
              │ Service: service-   │
              │       2048          │
              │      NodePort       │
              └──────────┬──────────┘
                         │
                         ▼
              ┌─────────────────────┐
              │    AWS Fargate      │
              │                     │
              │   2048 Pods         │
              └─────────────────────┘
```

## 🎮 Application

Open the ALB DNS name in a browser:

```text
http://<ALB-DNS-NAME>
```

and enjoy the game! 🎮
