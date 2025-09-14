pipeline {
  agent any
  environment {
    PATH       = "/usr/bin:/usr/local/bin:/usr/sbin:/usr/bin:/bin"
    AWS_REGION = 'us-east-2'
    APP_NAME   = 'hello-web'
    CLUSTER    = 'eks-demo2'
    IMAGE_TAG  = "${env.BUILD_NUMBER}"
  }
  options { timestamps() }
  stages {
    stage('Checkout'){ steps{ checkout scm } }
    stage('Discover AWS & Login to ECR'){
      steps{ sh '''
        set -eu
        ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
        printf "export ACCOUNT_ID=%s\nexport ECR_URI=%s.dkr.ecr.%s.amazonaws.com/%s\n"           "$ACCOUNT_ID" "$ACCOUNT_ID" "$AWS_REGION" "$APP_NAME" > .env.aws
        aws ecr describe-repositories --repository-names "${APP_NAME}" --region "${AWS_REGION}" >/dev/null 2>&1 ||           aws ecr create-repository --repository-name "${APP_NAME}" --region "${AWS_REGION}"
        aws ecr get-login-password --region "${AWS_REGION}" | docker login --username AWS --password-stdin           "${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
      ''' }
    }
    stage('Build & Smoke Test'){
      steps{ sh '''
        set -eu
        . ./.env.aws
        docker build -t "${APP_NAME}:${IMAGE_TAG}" .
        docker run -d --rm --name smoke -p 3000:3000 "${APP_NAME}:${IMAGE_TAG}"
        for i in $(seq 1 10); do curl -sf http://localhost:3000/ >/dev/null && break || sleep 1; done
        curl -sf http://localhost:3000/ | head -n 1
        docker rm -f smoke
      ''' }
    }
    stage('Tag & Push to ECR'){
      steps{ sh '''
        set -eu
        . ./.env.aws
        docker tag "${APP_NAME}:${IMAGE_TAG}" "${ECR_URI}:${IMAGE_TAG}"
        docker tag "${APP_NAME}:${IMAGE_TAG}" "${ECR_URI}:latest"
        docker push "${ECR_URI}:${IMAGE_TAG}"
        docker push "${ECR_URI}:latest"
      ''' }
    }
    stage('Configure kubectl for EKS'){
      steps{ sh '''
        set -eu
        which kubectl
        kubectl version --client
        aws eks update-kubeconfig --name "${CLUSTER}" --region "${AWS_REGION}"
        kubectl get nodes
      ''' }
    }
    stage('Deploy to EKS'){
      steps{ sh '''
        set -eu
        . ./.env.aws
        kubectl apply -f namespace.yaml
        kubectl apply -f deployment.yaml
        kubectl apply -f service.yaml
        kubectl -n demo set image deployment/hello-web hello-web="${ECR_URI}:${IMAGE_TAG}" --record=true
        kubectl -n demo rollout status deployment/hello-web --timeout=180s
        for i in $(seq 1 30); do
          H=$(kubectl -n demo get svc hello-web -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' 2>/dev/null || true)
          if [ -n "$H" ]; then echo "App URL: http://$H/"; curl -sf "http://$H/" | head -n 1 || true; exit 0; fi
          sleep 10
        done
        echo "Service created; ELB not ready yet. Check later: kubectl -n demo get svc hello-web"
      ''' }
    }
  }
  post { always { sh 'docker image prune -f || true' } }
}
