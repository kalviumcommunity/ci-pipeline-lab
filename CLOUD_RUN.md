# Cloud Run Pipeline Mapping Note

This document maps our current **Docker Hub + Docker Compose** CI/CD pipeline in GitHub Actions to an enterprise production setup target using **Google Cloud Platform (GCP) Artifact Registry and Cloud Run**.

---

## Conceptual & Architectural Mapping

| CI/CD Pipeline Phase | Docker Hub + Docker Compose (Current Lab) | Google Cloud Platform (Cloud Run Production) |
| :--- | :--- | :--- |
| **Authentication (`auth`)** | `docker/login-action` using `DOCKERHUB_USER` and `DOCKERHUB_TOKEN` secrets. | `google-github-actions/auth` using Workload Identity Federation or GCP Service Account JSON key stored in GitHub repository secrets (`GCP_SA_KEY`). |
| **Container Registry (`push`)** | Build and push image to Docker Hub repository: `<DOCKERHUB_USER>/app:${{ github.sha }}` using `docker/build-push-action`. | Build and push image to Google Artifact Registry: `LOCATION-docker.pkg.dev/PROJECT_ID/REPOSITORY/app:${{ github.sha }}` using `docker/build-push-action` or `gcloud builds`. |
| **Deployment (`deploy`)** | Execute local/runner deployment using Docker Compose: `IMAGE=... docker compose up -d`. | Deploy container image to serverless Cloud Run instance using `google-github-actions/deploy-cloudrun` or `gcloud run deploy`. |
| **Runtime & Scaling** | Single host local Docker daemon runtime. | Fully managed serverless container runtime with auto-scaling, HTTPS endpoints, and IAM control. |

---

## Example GitHub Actions Workflow snippet for Cloud Run

If migrating this workflow to GCP Cloud Run, the GitHub Actions configuration would look as follows:

```yaml
name: Build and Deploy to Cloud Run

on:
  push:
    branches:
      - main

env:
  PROJECT_ID: 'your-gcp-project-id'
  GAR_LOCATION: 'us-central1'
  REPOSITORY: 'my-docker-repo'
  SERVICE: 'my-web-app'
  REGION: 'us-central1'

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      # 1. AUTHENTICATION
      - name: Authenticate to GCP
        uses: google-github-actions/auth@v2
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}

      - name: Set up Cloud SDK
        uses: google-github-actions/setup-gcloud@v2

      - name: Authorize Docker for Artifact Registry
        run: |
          gcloud auth configure-docker ${{ env.GAR_LOCATION }}-docker.pkg.dev --quiet

      # 2. BUILD & PUSH TO ARTIFACT REGISTRY
      - name: Build and Push Container Image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ env.GAR_LOCATION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/${{ env.REPOSITORY }}/${{ env.SERVICE }}:${{ github.sha }}

  deploy-to-cloud-run:
    needs: build-and-push
    runs-on: ubuntu-latest
    steps:
      - name: Authenticate to GCP
        uses: google-github-actions/auth@v2
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}

      # 3. DEPLOY TO CLOUD RUN
      - name: Deploy to Cloud Run
        uses: google-github-actions/deploy-cloudrun@v2
        with:
          service: ${{ env.SERVICE }}
          region: ${{ env.REGION }}
          image: ${{ env.GAR_LOCATION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/${{ env.REPOSITORY }}/${{ env.SERVICE }}:${{ github.sha }}
          flags: '--allow-unauthenticated --port=8080'
```

---

## Key Benefits of Cloud Run Pipeline Migration
- **Immutable Artifacts**: Every commit builds a unique container tag stored in Artifact Registry.
- **Zero Infrastructure Management**: Cloud Run automatically manages server provisioning, network security, and TLS certificates.
- **Secure Service Account Access**: Uses fine-grained IAM roles rather than standard user credentials.
