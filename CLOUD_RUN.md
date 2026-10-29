# Mapping Local/Docker Hub CI Pipeline to Google Cloud Run

This document outlines how the current GitHub Actions CI/CD pipeline (targeting Docker Hub and local Docker Compose) maps to a production deployment on **Google Cloud Run**.

---

## 1. Pipeline Architecture Comparison

| Pipeline Stage | Current Pipeline (Docker Hub + Local) | Target Cloud Pipeline (Google Cloud Run) |
| :--- | :--- | :--- |
| **Authentication** | Docker Hub personal access token (`DOCKERHUB_USER`, `DOCKERHUB_TOKEN`) via `docker/login-action` | GCP Service Account Key (`GCP_SA_KEY`) or Workload Identity Federation via `google-github-actions/auth` |
| **Image Registry** | Docker Hub (`docker.io/<username>/app:<sha>`) | Google Artifact Registry (`<region>-docker.pkg.dev/<project>/<repo>/app:<sha>`) |
| **Image Tagging** | `<username>/app:${{ github.sha }}` (immutable commit SHA) | `<region>-docker.pkg.dev/<project-id>/<repo>/app:${{ github.sha }}` |
| **Deployment Stage** | `docker compose pull && docker compose up -d` | `gcloud run deploy` or `google-github-actions/deploy-cloudrun` |
| **Sequencing** | `deploy` job depends on `build` via `needs: build` | `deploy` job depends on `build` via `needs: build` |
| **Health Check** | `curl -f http://localhost:8080/health` | Automated Cloud Run startup/liveness probes or curling the deployed Service URL |

---

## 2. Key Components & Implementation Details

### A. Authentication: Service Account & Workload Identity
Instead of username/password or personal access tokens, GCP uses IAM Service Accounts:
- **Service Account Permissions**:
  - `roles/artifactregistry.writer`: Allows pushing container images to Google Artifact Registry.
  - `roles/run.admin`: Allows managing and deploying Cloud Run services.
  - `roles/iam.serviceAccountUser`: Allows deploying services that run under a designated runtime service account.
- **In GitHub Actions**:
  ```yaml
  - name: Authenticate to Google Cloud
    uses: google-github-actions/auth@v2
    with:
      credentials_json: ${{ secrets.GCP_SA_KEY }}
  ```
  *(Alternatively, Workload Identity Federation can be used to eliminate long-lived service account keys).*

---

### B. Registry: Google Artifact Registry
Docker Hub is replaced with Artifact Registry for regional proximity, managed security, and IAM control:
- Configure Docker authentication to Artifact Registry:
  ```yaml
  - name: Set up Cloud SDK & Docker Auth
    run: |
      gcloud auth configure-docker ${{ env.REGION }}-docker.pkg.dev --quiet
  ```
- Build and push immutable tag:
  ```yaml
  - name: Build and Push to Artifact Registry
    uses: docker/build-push-action@v5
    with:
      context: .
      push: true
      tags: ${{ env.REGION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/${{ env.REPOSITORY }}/app:${{ github.sha }}
  ```

---

### C. Deployment: `gcloud run deploy`
Instead of running `docker compose` on a self-managed host, the image is deployed as a managed serverless service:
- Deployment step:
  ```yaml
  - name: Deploy to Cloud Run
    run: |
      gcloud run deploy ci-pipeline-service \
        --image ${{ env.REGION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/${{ env.REPOSITORY }}/app:${{ github.sha }} \
        --region ${{ env.REGION }} \
        --platform managed \
        --allow-unauthenticated \
        --port 8080
  ```
- Or using the dedicated GitHub Action:
  ```yaml
  - name: Deploy to Cloud Run
    uses: google-github-actions/deploy-cloudrun@v2
    with:
      service: ci-pipeline-service
      image: ${{ env.REGION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/${{ env.REPOSITORY }}/app:${{ github.sha }}
      region: ${{ env.REGION }}
  ```

---

## 3. Example Cloud Run Workflow (`.github/workflows/deploy-cloudrun.yml.example`)

```yaml
name: Build and Deploy to Cloud Run

on:
  push:
    branches:
      - main

env:
  PROJECT_ID: ${{ secrets.GCP_PROJECT_ID }}
  REGION: us-central1
  REPOSITORY: ci-images
  SERVICE_NAME: ci-pipeline-app

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v2
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Configure Docker for Artifact Registry
        run: |
          gcloud auth configure-docker ${{ env.REGION }}-docker.pkg.dev --quiet

      - name: Build and push image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ env.REGION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/${{ env.REPOSITORY }}/app:${{ github.sha }}

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v2
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}

      - name: Deploy to Google Cloud Run
        uses: google-github-actions/deploy-cloudrun@v2
        with:
          service: ${{ env.SERVICE_NAME }}
          image: ${{ env.REGION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/${{ env.REPOSITORY }}/app:${{ github.sha }}
          region: ${{ env.REGION }}
          flags: '--allow-unauthenticated --port=8080'

      - name: Verify Health Endpoint
        run: |
          SERVICE_URL=$(gcloud run services describe ${{ env.SERVICE_NAME }} --region ${{ env.REGION }} --format 'value(status.url)')
          curl -f "$SERVICE_URL/health"
```
