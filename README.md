# EKS + Docker + Jenkins CI/CD (my notes, step-by-step)

This is the exact path I followed to get a tiny Node.js app building in **Jenkins**, pushed to **ECR**, and deployed on **EKS**. It’s copy‑paste friendly. 

> **My setup**
> - **Region:** `us-east-2` (Ohio)
> - **Ubuntu:** 22.04 LTS on both boxes
> - **Cluster:** `eks-demo3` (Kubernetes 1.29)
> - **Node group:** `ng-1` with `t3.large` (2 nodes)
> - **Node root volume:** **8 GiB** (explicitly set — small on purpose)
> - **App / image repo:** `hello-web`
> - **Git repo:** `EKS-Docker-Jenkins-Deployment` (this repo)

---

## What we’re building

- A single‑container Node.js app packaged with **Docker**.
- Image stored in **Amazon ECR**.
- **EKS** cluster created with **eksctl**.
- A **Jenkins** pipeline that:
  1) checks out this repo,
  2) builds + smoke‑tests the image,
  3) pushes to ECR,
  4) points `kubectl` at the cluster,
  5) applies K8s manifests and rolls out,
  6) prints a public **LoadBalancer URL**.

---

## Quick “why these services?” (one‑liners)

- **EC2** – VMs for my **Workbench** (ops box) and **Jenkins** server.
- **EKS** – managed Kubernetes control plane so I’m not running my own masters.
- **eksctl** – the CLI that actually stands up the EKS infra via CloudFormation for me.
- **ECR** – private Docker registry to store/pull the app image.
- **IAM** – permissions glue so EC2/EKS/ECR can talk to each other safely.
- **ELB** – the public endpoint I get when I use a `Service` of type `LoadBalancer`.
- **Jenkins** – CI/CD engine that turns a Git push into a running Deployment on EKS.
- **Docker** – builds/runs the app locally and in the pipeline.
- **kubectl** – CLI to interact with the Kubernetes API.
- **git/GitHub** – source of truth; Jenkins pulls all code from here.
- **CloudFormation** – what `eksctl` uses under the hood to create AWS resources.
- **jq/curl** – tiny helpers for scripting and smoke tests.

---

## Repo layout

```
.
├── Dockerfile
├── Jenkinsfile
├── namespace.yaml
├── deployment.yaml
├── service.yaml
├── package.json
├── server.js
└── .env.aws            # lives at repo ROOT (important)
```

**`.env.aws`** — the pipeline reads this so I don’t hard‑code env:
```dotenv
AWS_REGION=us-east-2
CLUSTER=eks-demo3
APP_NAME=hello-web
```

> If Jenkins ever says `.env.aws` is missing, make sure it’s truly at repo **root** (same level as `Jenkinsfile`). Wiping the Jenkins workspace and rebuilding also helps. I also included a safeguard stage in the Jenkinsfile that creates a fallback file if it’s missing.

---

## 0) Spin up two EC2 instances (Ubuntu 22.04)

I created two boxes:
- **Workbench** – where I run CLIs (aws/eksctl/kubectl) and cluster admin commands
- **Jenkins** – where the CI/CD server runs

**Security groups** I opened:
- Jenkins: TCP **8080** (from my IP)
- SSH: TCP **22** (from my IP)

**IAM role** on both instances: an admin‑ish role, or at least the ability to do `eks:*`, `cloudformation:*`, `ec2:*`, `ecr:*`, and pass the nodegroup role.

---

## 1) Prepare the Workbench (Ubuntu 22.04)

```bash
# Base
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release git jq unzip

# AWS CLI v2 (if not already on the AMI)
if ! command -v aws >/dev/null; then
  curl -sSL "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o /tmp/awscliv2.zip
  unzip -q /tmp/awscliv2.zip -d /tmp
  sudo /tmp/aws/install
fi

# Docker
sudo apt install -y docker.io
sudo usermod -aG docker $USER
newgrp docker <<'EOF'
docker ps >/dev/null && echo "Docker OK on Workbench"
EOF

# kubectl (match EKS minor: 1.29; architecture = amd64 on EC2)
curl -LO "https://dl.k8s.io/release/v1.29.15/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
rm kubectl
kubectl version --client

# eksctl
curl -sSL "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz" -o /tmp/eksctl.tgz
sudo tar -xzf /tmp/eksctl.tgz -C /usr/local/bin eksctl
eksctl version
```

> **Gotcha we hit:** accidentally downloading `arm64` binaries on an `amd64` instance makes `kubectl` error with “newline unexpected” or similar. Use `linux/amd64` URLs on typical x86_64 EC2.

---

## 2) Create the EKS cluster (with 8 GiB node volumes)

On the **Workbench**:
```bash
aws configure set default.region us-east-2

eksctl create cluster \
  --name eks-demo3 \
  --version 1.29 \
  --region us-east-2 \
  --nodegroup-name ng-1 \
  --node-type t3.large \
  --nodes 2 \
  --node-volume-size 8
```

Then wire up kubeconfig and verify:
```bash
aws eks update-kubeconfig --name eks-demo3 --region us-east-2
kubectl get nodes -o wide
```

> **Note on 8 GiB:** this is intentionally small to keep the demo lean. If you plan to run bigger images or lots of pods, bump this (EKS default is 80 GiB).

---

## 3) Prepare the Jenkins server (Ubuntu 22.04)

```bash
# Java + Jenkins repo
sudo apt update
sudo apt install -y fontconfig openjdk-17-jre

curl -fsSL https://pkg.jenkins.io/debian/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update
sudo apt install -y jenkins docker.io git curl jq unzip

# Let Jenkins use Docker
sudo usermod -aG docker jenkins
sudo systemctl enable --now docker jenkins

# AWS CLI v2 (if needed)
if ! command -v aws >/dev/null; then
  curl -sSL "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o /tmp/awscliv2.zip
  unzip -q /tmp/awscliv2.zip -d /tmp
  sudo /tmp/aws/install
fi

# kubectl client (same 1.29 amd64)
curl -LO "https://dl.k8s.io/release/v1.29.15/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
rm kubectl
kubectl version --client
```

Open Jenkins at: `http://<JENKINS_PUBLIC_IP>:8080`  
Unlock with `/var/lib/jenkins/secrets/initialAdminPassword` → install suggested plugins → create admin user.

---

## 4) Make sure the repo has the right files

This repo already includes:
- `Dockerfile`, `server.js`, `package.json`
- `namespace.yaml`, `deployment.yaml`, `service.yaml`
- `Jenkinsfile`
- `.env.aws` (at repo **root**)

`.env.aws` content I use:
```dotenv
AWS_REGION=us-east-2
CLUSTER=eks-demo3
APP_NAME=hello-web
```

---

## 5) Create the Jenkins Pipeline job

In Jenkins:
- **New Item → Pipeline**
- Name: `hello-web-deploy`
- **Pipeline script from SCM**
  - SCM: **Git**
  - Repo URL: `https://github.com/<your-username>/EKS-Docker-Jenkins-Deployment.git`
  - Branch: `*/main`
  - Script Path: `Jenkinsfile`
- Save → **Build Now**

Here’s the **final Jenkinsfile** I use (has a guard to create `.env.aws` if missing):

```groovy
pipeline {
  agent any
  options { timestamps() }

  stages {
    stage('Checkout') { steps { checkout scm } }

    stage('Ensure env file') {
      steps {
        sh '''
          set -eu
          if [ ! -f ./.env.aws ]; then
            cat > ./.env.aws <<EOF
AWS_REGION=us-east-2
CLUSTER=eks-demo3
APP_NAME=hello-web
EOF
            echo "Created fallback .env.aws (commit it to your repo to keep permanent)."
          fi
          echo "---- .env.aws ----"
          cat ./.env.aws
        '''
      }
    }

    stage('Discover AWS & Login to ECR') {
      steps {
        sh '''
          set -eu
          ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
          echo "ECR: ${ACCOUNT_ID}.dkr.ecr.us-east-2.amazonaws.com/hello-web"
          aws ecr describe-repositories --repository-names hello-web --region us-east-2 || \
            aws ecr create-repository --repository-name hello-web \
              --image-scanning-configuration scanOnPush=true --region us-east-2
          aws ecr get-login-password --region us-east-2 | \
            docker login --username AWS --password-stdin ${ACCOUNT_ID}.dkr.ecr.us-east-2.amazonaws.com
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
          for i in $(seq 1 10); do curl -sf http://localhost:3000/ && break || sleep 1; done
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
          ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
          ECR_URI="${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${APP_NAME}"
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
          . ./.env.aws
          aws eks update-kubeconfig --name "${CLUSTER}" --region "${AWS_REGION}"
          kubectl get nodes -o wide
        '''
      }
    }

    stage('Deploy to EKS') {
      steps {
        sh '''
          set -eu
          . ./.env.aws
          ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
          ECR_URI="${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${APP_NAME}"

          kubectl get ns demo >/dev/null 2>&1 || kubectl create ns demo
          kubectl apply -f namespace.yaml
          kubectl apply -f deployment.yaml
          kubectl apply -f service.yaml

          kubectl -n demo set image deployment/${APP_NAME} ${APP_NAME}="${ECR_URI}:1"
          kubectl -n demo rollout status deployment/${APP_NAME} --timeout=180s
          kubectl -n demo get deploy,rs,pods -o wide
          kubectl -n demo get svc ${APP_NAME} -o wide

          ELB=$(kubectl -n demo get svc ${APP_NAME} -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
          echo "APP URL: http://${ELB}/"
        '''
      }
    }
  }

  post {
    always { sh 'docker image prune -f || true' }
  }
}
```

---

## 6) First run + get the URL

Kick off **Build Now**. At the end, the console shows something like:
```
APP URL: http://a1b2c3d4e5f6-1234567890.us-east-2.elb.amazonaws.com/
```

You can also grab it from the Workbench:
```bash
aws eks update-kubeconfig --name eks-demo3 --region us-east-2
ELB=$(kubectl -n demo get svc hello-web -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "APP URL: http://${ELB}/"
curl -s "http://${ELB}/" | head -n1
```

---

## 7) Make a change and watch it redeploy

Edit `server.js` (change the message), commit to `main`, run the pipeline again (or wire up a GitHub webhook to auto‑trigger). Check rollout:
```bash
kubectl -n demo rollout status deployment/hello-web
```

---

## Troubleshooting (the real fixes I had to make)

- **Wrong `kubectl` binary** → Downloaded an HTML page/arm64 by mistake → fixed with the exact `linux/amd64` URL above.
- **Cluster name mismatch** → I renamed to `eks-demo3`; the pipeline reads `CLUSTER` from `.env.aws` so I don’t hard‑code it.
- **`.env.aws` missing** → Make sure it’s at repo root. If Jenkins cached an old workspace, wipe `/var/lib/jenkins/workspace/<job>` and rebuild. The Jenkinsfile also creates a fallback file.
- **Jenkins workspace confusion** → The job folder was `hello-web-deploy` (not `hello-web-deploy-2`). Listing `/var/lib/jenkins/workspace` cleared it up.
- **ELB pending** → Just needed a couple of minutes. Default `eksctl` VPC setup handles public LB fine.
- **Node image pulls** → Ensure node IAM role has `AmazonEC2ContainerRegistryReadOnly` policy attached.

---

## Day‑2 handy commands

Scale:
```bash
kubectl -n demo scale deploy/hello-web --replicas=3
kubectl -n demo get pods -o wide
```

Rollback:
```bash
kubectl -n demo rollout undo deploy/hello-web
```

Tail logs:
```bash
POD=$(kubectl -n demo get pods -l app=hello-web -o jsonpath='{.items[0].metadata.name}')
kubectl -n demo logs -f "$POD"
```

List ECR tags:
```bash
aws ecr list-images --repository-name hello-web --region us-east-2 \
  --query 'imageIds[].imageTag' --output text
```

Webhook (auto‑build on push):
- Jenkins job → **Configure → Build Triggers → GitHub hook trigger for GITScm polling**
- GitHub → **Settings → Webhooks → Add webhook**
  - Payload URL: `http://<JENKINS_PUBLIC_IP>:8080/github-webhook/`
  - Content type: `application/json`
  - Events: **Just the push event**

---

## Cleanup

```bash
# App
kubectl delete ns demo

# Cluster
eksctl delete cluster --region us-east-2 --name eks-demo3
```

---

## Why each file exists (quick recap)

- **Dockerfile** – how to build the container.
- **server.js / package.json** – the tiny Node.js app.
- **namespace.yaml** – isolates app resources into `demo`.
- **deployment.yaml** – desired state (replicas/pod template).
- **service.yaml** – public entrypoint via `LoadBalancer`.
- **Jenkinsfile** – CI/CD stages from build to deploy.
- **.env.aws** – centralizes region/cluster/app for the pipeline.

---

If you follow this README verbatim, you’ll get a working URL at the end. If something’s off, the Troubleshooting section above covers the exact errors I hit and how I fixed them.
