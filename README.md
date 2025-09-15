# EKS Docker Jenkins CI/CD Project

This project sets up a complete CI/CD pipeline using **AWS EKS**, **Docker**, and **Jenkins**. The goal is to build, test, push, and deploy a containerized Node.js application onto an EKS cluster. Below are the detailed steps and explanations of the services used.

---

## Services Used

- **Amazon EKS (Elastic Kubernetes Service):** Manages Kubernetes control plane so clusters can be run without managing master nodes. Used for hosting the containerized application.
- **Amazon EC2 (Elastic Compute Cloud):** Provides virtual servers to run Jenkins and the Workbench.  
- **Amazon ECR (Elastic Container Registry):** Stores Docker images built by Jenkins before deploying to EKS.  
- **IAM (Identity and Access Management):** Provides roles and permissions for EC2 instances, Jenkins, and EKS.  
- **CloudFormation:** Automatically provisions underlying resources when creating clusters with `eksctl`.  
- **Docker:** Builds and runs container images for the Node.js app.  
- **Jenkins:** Automates CI/CD steps, including build, test, push, and deploy.  
- **kubectl:** CLI tool for interacting with the Kubernetes cluster.  
- **eksctl:** Simplifies creating and managing EKS clusters.  
- **AWS CLI:** Used to interact with AWS services like EKS, ECR, and IAM.

---

## Step 1: Set Up the Workbench EC2

The Workbench EC2 acts as the admin/control machine to create and manage the EKS cluster.

1. Launch an **Ubuntu 22.04 EC2 instance** (t3.medium or larger recommended).
2. SSH into the instance.
3. Install required tools:

   ```bash
   sudo apt update -y
   sudo apt install -y unzip git curl docker.io
   ```

4. Install AWS CLI v2:

   ```bash
   curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
   unzip awscliv2.zip
   sudo ./aws/install
   ```

5. Install `eksctl`:

   ```bash
   curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz"
   tar -xzf eksctl_$(uname -s)_amd64.tar.gz -C /tmp
   sudo mv /tmp/eksctl /usr/local/bin
   ```

6. Install `kubectl`:

   ```bash
   curl -LO "https://dl.k8s.io/release/v1.34.0/bin/linux/amd64/kubectl"
   chmod +x kubectl
   sudo mv kubectl /usr/local/bin/
   ```

7. Verify installations:

   ```bash
   aws --version
   eksctl version
   kubectl version --client
   docker --version
   git --version
   ```

---

## Step 2: Create the EKS Cluster

Use `eksctl` from the Workbench to create the cluster:

```bash
eksctl create cluster \
  --name eks-demo3 \
  --version 1.29 \
  --region us-east-2 \
  --nodegroup-name ng-1 \
  --node-type t3.large \
  --nodes 2 \
  --node-volume-size 8
```

- **Cluster Name:** `eks-demo3`  
- **Region:** `us-east-2`  
- **Nodegroup:** `ng-1` with 2 nodes  
- **Instance Type:** `t3.large`  
- **Node Volume Size:** `8GB`

Once created, update kubeconfig:

```bash
aws eks update-kubeconfig --region us-east-2 --name eks-demo3
kubectl get nodes -o wide
```

---

## Step 3: Set Up Jenkins Server

1. Launch another **Ubuntu 22.04 EC2 instance** for Jenkins.
2. Install Java and Jenkins:

   ```bash
   sudo apt update -y
   sudo apt install -y openjdk-17-jdk
   curl -fsSL https://pkg.jenkins.io/debian/jenkins.io-2023.key | sudo tee \
     /usr/share/keyrings/jenkins-keyring.asc > /dev/null
   echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
     https://pkg.jenkins.io/debian binary/ | sudo tee \
     /etc/apt/sources.list.d/jenkins.list > /dev/null
   sudo apt update -y
   sudo apt install -y jenkins docker.io
   ```

3. Start and enable Jenkins:

   ```bash
   sudo systemctl enable jenkins
   sudo systemctl start jenkins
   ```

4. Add Jenkins to the Docker group:

   ```bash
   sudo usermod -aG docker jenkins
   sudo systemctl restart jenkins
   ```

5. Verify Jenkins can use Docker:

   ```bash
   sudo -iu jenkins bash -lc 'docker ps'
   ```

---

## Step 4: Configure Jenkins

1. Access Jenkins at `http://<JENKINS-EC2-PUBLIC-IP>:8080`.  
2. Unlock Jenkins using the admin password from:

   ```bash
   sudo cat /var/lib/jenkins/secrets/initialAdminPassword
   ```

3. Install suggested plugins.
4. Create a new **Pipeline Job** in Jenkins.
5. Connect Jenkins to your GitHub repo (`EKS-Docker-Jenkins-Deployment`).

---

## Step 5: Project Files

The GitHub repository should contain:

- **Dockerfile** – Defines how the Node.js app is containerized.  
- **server.js** – Node.js web server.  
- **package.json** – Node.js dependencies.  
- **deployment.yaml** – Kubernetes Deployment manifest.  
- **service.yaml** – Kubernetes Service manifest.  
- **namespace.yaml** – Namespace definition (`demo`).  
- **.env.aws** – Environment variables:

  ```bash
  ACCOUNT_ID=914261932225
  AWS_REGION=us-east-2
  APP_NAME=hello-web
  ECR_URI=914261932225.dkr.ecr.us-east-2.amazonaws.com/hello-web
  ```

- **Jenkinsfile** – Defines pipeline stages:
  1. Checkout code  
  2. Authenticate to AWS & ECR  
  3. Build & smoke test Docker image  
  4. Push to ECR  
  5. Configure kubectl  
  6. Deploy to EKS  

---

## Step 6: Run the Pipeline

When the pipeline is triggered in Jenkins:

1. The app is built into a Docker image.  
2. The image is pushed to Amazon ECR.  
3. Jenkins uses `kubectl` to apply manifests (`deployment.yaml`, `service.yaml`).  
4. The app is deployed onto EKS under the `demo` namespace.  
5. A LoadBalancer service exposes the app externally.

Check deployment status:

```bash
kubectl -n demo get all
kubectl -n demo get svc hello-web
```

---

## Step 7: Access the Application

Once deployed, retrieve the external IP of the service:

```bash
kubectl -n demo get svc hello-web
```

Open the external IP in a browser to see:

```
Hello from EKS via Jenkins CI/CD! 🚀
```

---

## Conclusion

This project demonstrates how to build a fully automated CI/CD pipeline with Jenkins, Docker, and Kubernetes on AWS EKS.  
- Jenkins automates the pipeline.  
- Docker builds the application image.  
- ECR stores the image.  
- EKS hosts the containerized application.  

This end-to-end setup ensures a repeatable and scalable deployment workflow.
