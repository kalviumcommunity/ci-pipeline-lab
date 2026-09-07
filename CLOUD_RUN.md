# Cloud Run Deployment Guide

To adapt this pipeline for deploying to Google Cloud Run, follow these steps:

1. **Authentication (Service Account)**:
   Instead of Docker Hub credentials, set up Google Cloud authentication in GitHub Actions using `google-github-actions/auth`. You will need a Service Account JSON key (stored as a secret `GCP_CREDENTIALS`).

2. **Push to Artifact Registry**:
   Change the image registry from Docker Hub to Google Artifact Registry. You will need to authenticate Docker with Artifact Registry using `gcloud auth configure-docker`.
   The image tag should be updated to point to your Artifact Registry repository, e.g., `[REGION]-docker.pkg.dev/[PROJECT_ID]/[REPO_NAME]/app:${{ github.sha }}`.

3. **Deploy to Cloud Run**:
   Replace the local `docker compose up -d` deployment step with the `google-github-actions/deploy-cloudrun` action or use the gcloud CLI directly:
   
   ```yaml
   - name: Deploy to Cloud Run
     run: |
       gcloud run deploy app-service \
         --image [REGION]-docker.pkg.dev/[PROJECT_ID]/[REPO_NAME]/app:${{ github.sha }} \
         --region [REGION] \
         --platform managed \
         --allow-unauthenticated
   ```
