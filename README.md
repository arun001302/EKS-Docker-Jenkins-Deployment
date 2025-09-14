# EKS · Docker · Jenkins — Starter Repo (us-east-2, cluster: `eks-demo2`)

This repo contains a minimal Node.js web app, Dockerfile, Kubernetes manifests, and a Jenkins Pipeline
that builds → pushes to ECR → deploys to EKS. It’s pre-wired for **us-east-2** and **cluster `eks-demo2`**.

## Files
- `Dockerfile` — builds the container image
- `package.json`, `server.js` — tiny Node.js web app (outputs a greeting)
- `namespace.yaml` — creates the `demo` namespace
- `deployment.yaml` — K8s Deployment (image placeholder; Jenkins sets the final tag)
- `service.yaml` — LoadBalancer Service exposing the app on port 80
- `Jenkinsfile` — Jenkins pipeline (uses BUILD_NUMBER as the image tag)
- `.dockerignore`, `.gitignore` — tidy builds

## Jenkins job (Pipeline script from SCM)
- **Repo**: your GitHub repo URL (this folder)
- **Branch**: `main`
- **Script Path**: `Jenkinsfile`

The pipeline will:
1. Log in to ECR and (if missing) create the repo `hello-web`.
2. Build the image and run a local smoke test.
3. Push tags: `${BUILD_NUMBER}` and `latest`.
4. Configure kubectl for EKS (`eks-demo2` in `us-east-2`).
5. Apply manifests; set image to the pushed tag; wait for rollout.
6. Print the external URL (ELB hostname).

## Notes
- Change cluster/region by editing `CLUSTER`/`AWS_REGION` in `Jenkinsfile`.
- The node instance type (e.g., `t3.large`) is chosen when you create the nodegroup with `eksctl`
  and is not controlled by these files.
