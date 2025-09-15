pipeline {
  agent any
  options { timestamps() }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Ensure env file') {
      steps {
        sh '''
          set -eu
          # Create .env.aws if it’s missing (safety net)
          if [ ! -f ./.env.aws ]; then
            cat > ./.env.aws <<EOF
AWS_REGION=us-east-2
CLUSTER=eks-demo4
APP_NAME=hello-web
EOF
            echo "Created fallback .env.aws (commit this file to your repo to keep it permanent)."
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
          echo "export ACCOUNT_ID=${ACCOUNT_ID}"
          echo "export ECR_URI=${ACCOUNT_ID}.dkr.ecr.us-east-2.amazonaws.com/hello-web"
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
