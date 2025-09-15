# EKS + Docker + Jenkins CI/CD (Ubuntu 22.04, us-east-2)

A complete, reproducible walkthrough to build, test, push, and deploy a Node.js container to **Amazon EKS** using **Jenkins** CI/CD. These instructions assume two EC2 instances: a **Workbench** (admin box) and a **Jenkins** server, both running **Ubuntu 22.04** in **us-east-2**.

---

## Architecture Overview (What each service does)

- **Amazon EC2** – Virtual machines. Used for the Workbench (admin box) and the Jenkins server.
- **Amazon EKS** – Managed Kubernetes control plane. Runs the application on worker nodes.
- **Amazon ECR** – Private Docker registry to store images built by Jenkins.
- **AWS IAM** – Identity & access control. Grants EC2 instances (and Jenkins) permissions to call AWS APIs.
- **AWS CloudFormation** – Under the hood orchestration used by `eksctl` to provision cluster resources.
- **Docker** – Builds and runs the containerized Node.js app.
- **Jenkins** – CI/CD automation: builds, tests, pushes images to ECR, and deploys to EKS.
- **kubectl** – Command-line to interact with Kubernetes clusters.
- **eksctl** – CLI to create and manage EKS clusters easily.
- **AWS CLI** – CLI to interact with AWS services (EKS/ECR/IAM/etc.).

---

## Variables used throughout

```bash
# Use these consistently on both Workbench and Jenkins
export AWS_REGION=us-east-2
export CLUSTER=eks-demo3
export APP_NAME=hello-web
# Your AWS Account ID (will be populated later on the machines that have IAM role/credentials)
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text 2>/dev/null || echo "<your-account-id>")
export ECR_URI="${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${APP_NAME}"
```

> **Tip:** On EC2 instances with an attached IAM role, `ACCOUNT_ID=$(aws sts get-caller-identity …)` will fill automatically.

---

## 1) Provision EC2 Instances (Console)

Create **two** EC2 instances in **us-east-2** (Ohio), Ubuntu 22.04 (amd64):
- **Workbench**: type `t3.medium` (or larger), for admin/cluster creation.
- **Jenkins**:  type `t3.medium` (or larger), for CI server.

### 1.1 Security Groups
- **Workbench SG**
  - Inbound: TCP 22 (SSH) from your IP
  - Outbound: allow all (default)
- **Jenkins SG**
  - Inbound: TCP 22 (SSH) from your IP
  - Inbound: TCP 8080 (Jenkins UI) from your IP (or corporate range)
  - Outbound: allow all (default)

### 1.2 IAM Role (Instance Profile)
For a simple lab, attach a role with **broad** permissions (you can tighten later):
- **Role name**: `EC2AdminRole` (example)
- **Policy**: `AdministratorAccess` (lab/demo only)

> Minimum set (if you prefer restrictive access): EKS (cluster + node), ECR (pull/push), EC2 (networking/ENI/SG), IAM PassRole for EKS, CloudFormation, ELB/ALB, CloudWatch logs.

Attach this **role** to both **Workbench** and **Jenkins** instances at launch.

### 1.3 Key Pair & Networking
- Pick or create a Key Pair to SSH.
- Place both instances in the same VPC (default VPC is fine).

---

## 2) Prepare the Workbench (Ubuntu 22.04)

SSH into the **Workbench** and run:

```bash
sudo apt update -y
sudo apt install -y unzip git curl docker.io jq

# AWS CLI v2
curl -sSL "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
unzip -q awscliv2.zip
sudo ./aws/install

# eksctl (latest)
curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz"
tar -xzf eksctl_$(uname -s)_amd64.tar.gz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin/

# kubectl (match EKS 1.29)
curl -sLO "https://amazon-eks.s3.us-east-2.amazonaws.com/1.29.0/2024-08-08/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/kubectl
sudo chown root:root /usr/local/bin/kubectl

# Verify
aws --version
eksctl version
kubectl version --client
docker --version
git --version
jq --version

# Confirm IAM identity from instance role
aws sts get-caller-identity
```

> **Note:** The kubectl URL must match your CPU architecture (Ubuntu EC2 is amd64). If you ever see `Syntax error: newline unexpected`, you likely downloaded an HTML error page or the wrong architecture. Re-download using the **amd64** URL above.

---

## 3) Create the EKS Cluster (t3.large, 2 nodes, 8GB volumes)

From the **Workbench**:

```bash
export AWS_REGION=us-east-2
export CLUSTER=eks-demo3

eksctl create cluster \
  --name "${CLUSTER}" \
  --version 1.29 \
  --region "${AWS_REGION}" \
  --nodegroup-name ng-1 \
  --node-type t3.large \
  --nodes 2 \
  --node-volume-size 8
```

When it finishes, wire up `kubectl` and verify nodes:

```bash
aws eks update-kubeconfig --name "${CLUSTER}" --region "${AWS_REGION}"
kubectl get nodes -o wide
```

---

## 4) Create an ECR Repository

Still on the **Workbench** (or console UI):

```bash
export APP_NAME=hello-web
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export AWS_REGION=us-east-2

aws ecr create-repository --repository-name "${APP_NAME}" --region "${AWS_REGION}" || true

export ECR_URI="${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${APP_NAME}"
echo "${ECR_URI}"
```

---

## 5) Prepare the Jenkins Server

SSH into the **Jenkins** EC2 and install prerequisites:

```bash
sudo apt update -y
sudo apt install -y openjdk-17-jdk docker.io git curl unzip

# Jenkins repo & install
curl -fsSL https://pkg.jenkins.io/debian/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc >/dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian binary/ | \
  sudo tee /etc/apt/sources.list.d/jenkins.list >/dev/null
sudo apt update -y
sudo apt install -y jenkins

# AWS CLI v2
curl -sSL "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
unzip -q awscliv2.zip
sudo ./aws/install

# kubectl (client for 1.29, amd64)
curl -sLO "https://amazon-eks.s3.us-east-2.amazonaws.com/1.29.0/2024-08-08/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/kubectl
sudo chown root:root /usr/local/bin/kubectl

# Give Jenkins access to Docker
sudo usermod -aG docker jenkins
sudo systemctl enable --now docker jenkins

# Verify Jenkins can talk to Docker (new shell for jenkins)
sudo -iu jenkins bash -lc 'docker ps || id'
```

Retrieve the initial admin password and open the UI:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
# Browse to: http://<JENKINS_PUBLIC_IP>:8080
# Complete the setup wizard (Suggested plugins are fine).
```

---

## 6) Repository Layout (GitHub)

Create or use a repo named **`EKS-Docker-Jenkins-Deployment`** with the following files at the **repo root**:

```
.
├── Dockerfile
├── server.js
├── package.json
├── namespace.yaml
├── deployment.yaml
├── service.yaml
├── Jenkinsfile
└── .env.aws
```

### 6.1 Dockerfile
```dockerfile
FROM node:20-alpine
WORKDIR /usr/src/app
COPY package*.json ./
RUN npm ci --omit=dev || npm install --omit=dev
COPY server.js ./
EXPOSE 3000
CMD ["npm","start"]
```

### 6.2 server.js
```js
const http = require('http');
const hostname = '0.0.0.0';
const port = 3000;

const server = http.createServer((req, res) => {
  res.statusCode = 200;
  res.setHeader('Content-Type', 'text/plain');
  res.end('Hello from EKS via Jenkins CI/CD! 🚀\n');
});

server.listen(port, hostname, () => {
  console.log(`Server running at http://${hostname}:${port}/`);
});
```

### 6.3 package.json
```json
{
  "name": "hello-web",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {}
}
```

### 6.4 namespace.yaml
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo
```

### 6.5 deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-web
  namespace: demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello-web
  template:
    metadata:
      labels:
        app: hello-web
    spec:
      containers:
        - name: hello-web
          image: <ECR_URI>:latest
          ports:
            - containerPort: 3000
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 128Mi
```

> Replace `<ECR_URI>` with your value, for example:
> `914261932225.dkr.ecr.us-east-2.amazonaws.com/hello-web`

### 6.6 service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-web
  namespace: demo
spec:
  type: LoadBalancer
  selector:
    app: hello-web
  ports:
    - port: 80
      targetPort: 3000
```

### 6.7 .env.aws
```bash
ACCOUNT_ID=914261932225
AWS_REGION=us-east-2
APP_NAME=hello-web
ECR_URI=914261932225.dkr.ecr.us-east-2.amazonaws.com/hello-web
```

### 6.8 Jenkinsfile
```groovy
pipeline {
  agent any

  environment {
    AWS_REGION = 'us-east-2'
    CLUSTER    = 'eks-demo3'
    APP_NAME   = 'hello-web'
  }

  options {
    timestamps()
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Discover AWS & Login to ECR') {
      steps {
        sh '''
          set -eu
          ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
          echo "export ACCOUNT_ID=${ACCOUNT_ID}"
          echo "export ECR_URI=${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${APP_NAME}"
          aws ecr describe-repositories --repository-names ${APP_NAME} --region ${AWS_REGION} >/dev/null 2>&1 || \
            aws ecr create-repository --repository-name ${APP_NAME} --region ${AWS_REGION}
          aws ecr get-login-password --region ${AWS_REGION} | \
            docker login --username AWS --password-stdin ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
        '''
      }
    }

    stage('Build & Smoke Test') {
      steps {
        sh '''
          set -eu
          . ./.env.aws
          docker build -t ${APP_NAME}:1 .
          docker run -d --rm --name smoke -p 3000:3000 ${APP_NAME}:1
          # Wait briefly for app to boot, then hit it
          for i in $(seq 1 10); do
            curl -sf http://localhost:3000/ && break || sleep 1
          done
          curl -sf http://localhost:3000/ | head -n 1
          docker rm -f smoke
        '''
      }
    }

    stage('Tag & Push to ECR') {
      steps {
        sh '''
          set -eu
          . ./.env.aws
          docker tag ${APP_NAME}:1 ${ECR_URI}:1
          docker tag ${APP_NAME}:1 ${ECR_URI}:latest
          docker push ${ECR_URI}:1
          docker push ${ECR_URI}:latest
        '''
      }
    }

    stage('Configure kubectl for EKS') {
      steps {
        sh '''
          set -eu
          which kubectl
          kubectl version --client
          aws eks update-kubeconfig --name ${CLUSTER} --region ${AWS_REGION}
          kubectl get nodes
        '''
      }
    }

    stage('Deploy to EKS') {
      steps {
        sh '''
          set -eu
          # Ensure namespace exists
          kubectl get ns demo >/dev/null 2>&1 || kubectl create ns demo

          # Apply manifests
          kubectl apply -f namespace.yaml
          kubectl apply -f deployment.yaml
          kubectl apply -f service.yaml

          # Patch deployment image to the just-pushed tag:latest
          . ./.env.aws
          kubectl -n demo set image deployment/hello-web hello-web=${ECR_URI}:latest

          # Wait for rollout
          kubectl -n demo rollout status deployment/hello-web --timeout=180s

          # Show service
          kubectl -n demo get svc hello-web -o wide
        '''
      }
    }
  }

  post {
    always {
      sh 'docker image prune -f || true'
    }
  }
}
```

> Ensure `.env.aws` is at the **repo root** so Jenkins sees it after `checkout scm`.

---

## 7) Create the Jenkins Pipeline

1. In Jenkins → **New Item** → **Pipeline** (name it e.g. `hello-web-deploy`).
2. Definition: **Pipeline script from SCM**.
3. SCM: **Git**, Repository URL: `https://github.com/<your-user>/EKS-Docker-Jenkins-Deployment.git`
4. Branch: `main`
5. Save → **Build Now**.

Jenkins will:
- Build the Docker image and smoke test locally.
- Push the image to ECR.
- Configure `kubectl` to the EKS cluster.
- Apply Kubernetes manifests and roll out the deployment.

---

## 8) Validate the Deployment

On the **Workbench** (or any box with kubeconfig to the cluster):

```bash
kubectl -n demo get all
kubectl -n demo get svc hello-web
```

Look for the **EXTERNAL-IP** of the `hello-web` service. Open it in a browser (port 80) and you should see:

```
Hello from EKS via Jenkins CI/CD! 🚀
```

---

## Troubleshooting (Most common fixes)

- **kubectl: `Syntax error: newline unexpected`**
  - Cause: downloaded wrong architecture or an HTML error page.
  - Fix: re-download using the **amd64** URL for Ubuntu EC2:
    ```bash
    curl -sLO "https://amazon-eks.s3.us-east-2.amazonaws.com/1.29.0/2024-08-08/bin/linux/amd64/kubectl"
    chmod +x kubectl && sudo mv kubectl /usr/local/bin/
    ```

- **Jenkins can’t run Docker (`permission denied`)**
  - Fix: `sudo usermod -aG docker jenkins && sudo systemctl restart jenkins`

- **Jenkins fails on `.env.aws: No such file`**
  - Ensure `.env.aws` is committed at the **repo root** and the job checks out that repo.

- **`No cluster found for name: eks-demo2`**
  - Ensure your cluster name matches in the Jenkinsfile (`CLUSTER=eks-demo3`).

- **CloudFormation stack already exists**
  - Use a different cluster name or delete the old stack:
    ```bash
    eksctl delete cluster --name eks-old --region us-east-2
    ```

- **Nodes not Ready**
  - Wait a few minutes after cluster creation; verify with `kubectl get nodes -o wide`.

---

## Clean Up

```bash
# From Workbench
eksctl delete cluster --name "${CLUSTER}" --region "${AWS_REGION}"

# Delete ECR images and repository (replace with your values)
aws ecr batch-delete-image \
  --repository-name "${APP_NAME}" \
  --image-ids imageTag=latest imageTag=1 \
  --region "${AWS_REGION}" || true
aws ecr delete-repository --repository-name "${APP_NAME}" --region "${AWS_REGION}" --force || true

# Terminate EC2 instances (Workbench & Jenkins) in the AWS console
```

---

## Notes & Recommendations

- The node volume size was intentionally set to **8GB** for a minimal demo.
- For production, use AL2023 or Bottlerocket nodes and enable logging/monitoring (CloudWatch, EKS add-ons).
- Replace `AdministratorAccess` with least-privilege IAM policies.
- Restrict Jenkins 8080 access to trusted IPs only or put it behind a reverse proxy with TLS.
