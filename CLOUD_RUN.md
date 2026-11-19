# Cloud Run Architecture & Deployment Mapping Note

This document outlines the architectural mapping and transition path from the local Docker Hub + Docker Compose CI/CD pipeline to an enterprise-grade cloud deployment on **Google Cloud Platform (GCP)** using **Google Artifact Registry** and **Google Cloud Run**.

---

## 1. Architectural Mapping Overview

In our baseline CI/CD pipeline:
* **Registry Auth:** Docker Hub username and personal access token via `docker/login-action`.
* **Container Registry & Push:** Docker Hub repository (`<dockerhub_user>/app:<sha>`).
* **Deploy Runtime:** Runner's local Docker daemon managed via `docker compose up -d`.
* **Health Check:** `curl http://localhost:8080/health`.

When promoting this pipeline to Google Cloud, the pipeline stages map directly to GCP managed services:

| CI/CD Stage | Local / Docker Hub Pipeline | Google Cloud Run Architecture | Key Benefit in Production |
| :--- | :--- | :--- | :--- |
| **Authentication** | Docker Hub credentials (`DOCKERHUB_USER`, `DOCKERHUB_TOKEN`) | **GCP Service Account** / **Workload Identity Federation (WIF)** | Short-lived tokens, least-privilege IAM, zero long-lived credentials |
| **Registry & Push** | Docker Hub (`<user>/app:${{ github.sha }}`) | **Google Artifact Registry (GAR)** (`<region>-docker.pkg.dev/<project>/<repo>/app:<sha>`) | Private, regional co-location with Cloud Run, vulnerability scanning |
| **Deploy Stage** | `docker compose up -d` (Local container) | **`gcloud run deploy`** (or `google-github-actions/deploy-cloudrun`) | Fully managed serverless, auto-scaling (including scale-to-zero), zero server ops |
| **Health Check & Verification** | `curl -i http://localhost:8080/health` | `curl -i https://<service-url>/health` | HTTPS termination, managed ingress, custom domains, native health probes |

---

## 2. The Three Core GCP Components

### A. Authentication: Service Account & Workload Identity Federation
Instead of static personal access tokens:
1. **Google Service Account (SA):** A dedicated service account (e.g. `github-deployer@<project-id>.iam.gserviceaccount.com`) is granted fine-grained IAM roles:
   * `roles/artifactregistry.writer`: Allows pushing images to Google Artifact Registry.
   * `roles/run.admin`: Allows creating and updating Cloud Run services and revisions.
   * `roles/iam.serviceAccountUser`: Allows binding the Cloud Run runtime identity.
2. **Workload Identity Federation (Recommended):** Replaces static JSON keys with temporary OIDC tokens generated directly between GitHub Actions and GCP IAM. Alternatively, a base64-encoded Service Account JSON Key can be stored in GitHub Secrets (`GCP_SA_KEY`).
3. **GitHub Action:** `google-github-actions/auth@v2`.

### B. Image Storage & Push: Google Artifact Registry (GAR)
Docker Hub is replaced with Artifact Registry:
1. **Repository URL Format:**
   ```text
   <REGION>-docker.pkg.dev/<PROJECT_ID>/<REPOSITORY_NAME>/app:${{ github.sha }}
   ```
   *(e.g., `us-central1-docker.pkg.dev/my-gcp-project/containers/app:c515a74`)*
2. **Docker Authentication to GAR:**
   ```yaml
   - name: Log in to Google Artifact Registry
     uses: docker/login-action@v3
     with:
       registry: <REGION>-docker.pkg.dev
       username: _json_key # or oauth2accesstoken via WIF
       password: ${{ secrets.GCP_SA_KEY }}
   ```
3. **Benefits:** Ultra-fast image pulls due to internal Google backbone network peering directly to Cloud Run instances, along with automatic vulnerability scanning.

### C. Deployment: Serverless Containers on Google Cloud Run
Local Docker Compose is replaced with Cloud Run deployment:
1. **CLI / Action Command:**
   ```bash
   gcloud run deploy app-service \
     --image <REGION>-docker.pkg.dev/<PROJECT_ID>/<REPOSITORY>/app:${{ github.sha }} \
     --region <REGION> \
     --platform managed \
     --allow-unauthenticated \
     --port 8080
   ```
   Or via the official `google-github-actions/deploy-cloudrun@v2` GitHub Action.
2. **Production Capabilities:**
   * **Auto-Scaling:** Dynamically scales container instances from 0 to 100+ based on incoming HTTP request volume.
   * **Revisions & Traffic Splitting:** Every deploy creates an immutable revision, enabling canary releases and instant rollbacks.
   * **Built-in TLS/HTTPS:** Managed HTTPS endpoints out of the box with automated certificate provisioning.

---

## 3. Reference Cloud Run GitHub Actions Workflow

Below is the production-ready YAML workflow (`.github/workflows/deploy-cloudrun.yml`) illustrating the GCP deployment pattern:

```yaml
name: Build and Deploy to Cloud Run

on:
  push:
    branches:
      - main

env:
  PROJECT_ID: ${{ secrets.GCP_PROJECT_ID }}
  GAR_LOCATION: us-central1
  GAR_REPOSITORY: containers
  SERVICE_NAME: ci-pipeline-app
  REGION: us-central1

jobs:
  build-and-deploy:
    name: Build, Push & Deploy to Cloud Run
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      # 1. GCP Authentication (Auth -> Service Account)
      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v2
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}

      - name: Set up Cloud SDK
        uses: google-github-actions/setup-gcloud@v2

      # 2. Configure Docker authentication for Artifact Registry
      - name: Configure Docker for Artifact Registry
        run: |
          gcloud auth configure-docker ${{ env.GAR_LOCATION }}-docker.pkg.dev --quiet

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      # 3. Build & Push Image (Push -> Artifact Registry)
      - name: Build and push image to Artifact Registry
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ env.GAR_LOCATION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/${{ env.GAR_REPOSITORY }}/app:${{ github.sha }}

      # 4. Deploy Image to Cloud Run (Deploy -> gcloud run deploy)
      - name: Deploy to Google Cloud Run
        id: deploy
        uses: google-github-actions/deploy-cloudrun@v2
        with:
          service: ${{ env.SERVICE_NAME }}
          region: ${{ env.REGION }}
          image: ${{ env.GAR_LOCATION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/${{ env.GAR_REPOSITORY }}/app:${{ github.sha }}
          flags: '--port=8080 --allow-unauthenticated'

      # 5. Service Verification
      - name: Verify Live Service Health
        run: |
          curl -if "${{ steps.deploy.outputs.url }}/health"
```

---

## 4. Key Differences & Operational Trade-offs

| Aspect | Local Compose | Google Cloud Run |
| :--- | :--- | :--- |
| **High Availability** | None (Single host, vulnerable to downtime) | High (Multi-zone redundant infrastructure) |
| **Zero-Downtime Deploys** | Requires proxy tricks or temporary drops | Native (Traffic routes to new revision only after health checks pass) |
| **Observability** | `docker logs` on the machine | Google Cloud Logging, Cloud Monitoring, Cloud Trace |
| **Cost** | Fixed server cost | Pay-per-use (Pay only for compute during active requests, scale to zero) |
