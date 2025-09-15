# CI/CD Pipeline for EKS with Jenkins and Docker

This repository provides a complete, step-by-step guide for building a CI/CD pipeline that automatically deploys a Node.js web application to Amazon EKS.

I created this project to document the entire process in a clear and copy-paste friendly format. It covers everything from provisioning infrastructure and setting up Jenkins to the final automated deployment.

### Project Environment
This setup was tested using the following configuration:
- **AWS Region:** `us-east-2` (Ohio)
- **Operating System:** Ubuntu 22.04 LTS
- **EKS Cluster:** `eks-demo3` (Kubernetes v1.29)
- **EKS Node Group:** `ng-1` with 2x `t3.large` instances
- **EC2 Root Volume:** 8 GiB
- **ECR Repository:** `hello-web`

---

## Project Overview

This pipeline automates the standard CI/CD workflow:
1.  A simple Node.js HTTP server is containerized using **Docker**.
2.  The resulting container image is pushed to a private **Amazon ECR** repository.
3.  An **Amazon EKS** cluster runs the application as a Kubernetes Deployment and exposes it via a Service.
4.  A **Jenkins** pipeline orchestrates the entire process: checking out code, building the image, running tests, pushing to ECR, and deploying to EKS.

### Technology Stack

Here’s a quick breakdown of each component and its role in the pipeline:

-   **EC2:** Hosts the **Workbench** (my CLI admin machine) and the **Jenkins** server.
-   **EKS:** Provides the managed Kubernetes control plane, so I don't have to manage master nodes.
-   **eksctl:** A simple CLI tool for creating and managing clusters on EKS.
-   **ECR:** A private and secure container registry for my Docker images.
-   **IAM:** Manages roles and policies to ensure secure communication between AWS services.
-   **ELB:** The internet-facing Application Load Balancer created by the Kubernetes `Service` to expose the app.
-   **Jenkins:** The CI/CD engine that automates the build, test, and deploy workflow.
-   **Docker:** Builds the container image for the application.
-   **kubectl:** The command-line tool for interacting with the Kubernetes cluster.
-   **Git/GitHub:** Provides source control for the application code and Kubernetes manifests.

---

## Repository Structure

The project is organized with the following file layout:

.
├── Dockerfile          # Defines the Node.js application container
├── Jenkinsfile         # The CI/CD pipeline-as-code script
├── namespace.yaml      # Kubernetes Namespace for the application
├── deployment.yaml     # Kubernetes Deployment manifest
├── service.yaml        # Kubernetes Service manifest (type: LoadBalancer)
├── package.json        # Node.js dependencies
├── server.js           # The simple Node.js web server
└── .env.aws            # Environment variables for the pipeline


The `.env.aws` file at the root of the repository is used to configure the pipeline:
```dotenv
AWS_REGION=us-east-2
CLUSTER=eks-demo3
APP_NAME=hello-web
Setup Instructions
Step 0: Provision EC2 Instances
I started by creating two EC2 instances:

Workbench: A machine for running aws, eksctl, and kubectl commands.

Jenkins: The CI/CD server that will run the pipeline.

I configured their security groups to allow the following inbound traffic:

Jenkins: TCP port 8080 from my IP address.

SSH: TCP port 22 from my IP address.

Both instances were assigned an IAM role with sufficient permissions to manage EKS, ECR, CloudFormation, and EC2.

Step 1: Prepare the Workbench Instance
On the Workbench instance, I installed all the required CLI tools.

Bash

# Install base tools
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release git jq unzip

# Install AWS CLI v2
if ! command -v aws >/dev/null; then
  curl -sSL "[https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip](https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip)" -o /tmp/awscliv2.zip
  unzip -q /tmp/awscliv2.zip -d /tmp
  sudo /tmp/aws/install
fi

# Install Docker
sudo apt install -y docker.io
sudo usermod -aG docker $USER
newgrp docker <<'EOF'
docker ps >/dev/null && echo "Docker configured successfully."
EOF

# Install kubectl for Kubernetes v1.29
curl -LO "[https://dl.k8s.io/release/v1.29.15/bin/linux/amd64/kubectl](https://dl.k8s.io/release/v1.29.15/bin/linux/amd64/kubectl)"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
rm kubectl
kubectl version --client

# Install eksctl
curl -sSL "[https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz](https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz)" -o /tmp/eksctl.tgz
sudo tar -xzf /tmp/eksctl.tgz -C /usr/local/bin eksctl
eksctl version
Step 2: Create the EKS Cluster
From the Workbench instance, I created the EKS cluster. The --node-volume-size=8 flag is important to prevent running out of disk space.

Bash

# Configure the default AWS region
aws configure set default.region us-east-2

# Create the cluster
eksctl create cluster \
  --name eks-demo3 \
  --version 1.29 \
  --region us-east-2 \
  --nodegroup-name ng-1 \
  --node-type t3.large \
  --nodes 2 \
  --node-volume-size 8
After the cluster was created, I configured kubectl to connect to it and verified the nodes were ready.

Bash

aws eks update-kubeconfig --name eks-demo3 --region us-east-2
kubectl get nodes -o wide
Step 3: Set Up the Jenkins Server
On the Jenkins instance, I installed Jenkins, Java, and the same CLI tools needed for the pipeline.

Bash

# Install Java and Jenkins
sudo apt update
sudo apt install -y fontconfig openjdk-17-jre
curl -fsSL [https://pkg.jenkins.io/debian/jenkins.io-2023.key](https://pkg.jenkins.io/debian/jenkins.io-2023.key) | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  [https://pkg.jenkins.io/debian](https://pkg.jenkins.io/debian) binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt update
sudo apt install -y jenkins

# Install Docker and other tools
sudo apt install -y docker.io git curl jq unzip

# Add the 'jenkins' user to the 'docker' group
sudo usermod -aG docker jenkins
sudo systemctl enable --now docker jenkins

# Install AWS CLI v2
if ! command -v aws >/dev/null; then
  curl -sSL "[https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip](https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip)" -o /tmp/awscliv2.zip
  unzip -q /tmp/awscliv2.zip -d /tmp
  sudo /tmp/aws/install
fi

# Install kubectl
curl -LO "[https://dl.k8s.io/release/v1.29.15/bin/linux/amd64/kubectl](https://dl.k8s.io/release/v1.29.15/bin/linux/amd64/kubectl)"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
rm kubectl
kubectl version --client
Next, I completed the Jenkins setup in the browser at http://<JENKINS_PUBLIC_IP>:8080. I retrieved the initial admin password from /var/lib/jenkins/secrets/initialAdminPassword, installed the suggested plugins, and created an admin user.

Step 4: Create the Jenkins Pipeline Job
In the Jenkins dashboard, I created the pipeline:

New Item → Pipeline

Named the job hello-web-deploy.

Under the Pipeline section, selected Pipeline script from SCM.

SCM: Git

Repository URL: https://github.com/<your-username>/<your-repo-name>.git

Branch Specifier: */main

Script Path: Jenkinsfile

Saved the pipeline and clicked Build Now.

Verifying the Deployment
After the first pipeline build succeeds, the console output will display the public URL for the application.

APP URL: [http://a1b2c3d4e5f6-1234567890.us-east-2.elb.amazonaws.com/](http://a1b2c3d4e5f6-1234567890.us-east-2.elb.amazonaws.com/)
I can also retrieve this URL from my Workbench at any time with kubectl.

Bash

aws eks update-kubeconfig --name eks-demo3 --region us-east-2
ELB=$(kubectl -n demo get svc hello-web -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "Application URL: http://${ELB}/"

# Test the endpoint
curl -s "http://${ELB}/" | head -n1
Common Operations
Here are some common commands I use for managing the application post-deployment.

Scale the deployment:

Bash

kubectl -n demo scale deploy/hello-web --replicas=3
Roll back to a previous version:

Bash

kubectl -n demo rollout undo deploy/hello-web
Tail the application logs:

Bash

POD=$(kubectl -n demo get pods -l app=hello-web -o jsonpath='{.items[0].metadata.name}')
kubectl -n demo logs -f "$POD"
List image tags in ECR:

Bash

aws ecr list-images --repository-name hello-web --region us-east-2 \
  --query 'imageIds[].imageTag' --output text
Cleanup
To avoid ongoing AWS charges, I run the following commands to tear down all the resources.

Delete the application from the cluster:

Bash

kubectl delete ns demo
Delete the EKS cluster and its associated resources:

Bash

eksctl delete cluster --region us-east-2 --name eks-demo3

***








