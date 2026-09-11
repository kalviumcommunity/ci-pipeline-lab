# Cloud Run CI/CD Pipeline Mapping

This document outlines how the current GitHub Actions pipeline (Docker Hub + Docker Compose) maps to a production deployment on **Google Cloud Run** using **Google Artifact Registry**.

---

## Architecture Mapping

| Component / Stage | Current Lab Pipeline (Docker Hub + Local) | Production Cloud Run Pipeline |
| :--- | :--- | :--- |
| **Authentication** | `docker/login-action` using Docker Hub Username & Access Token (`DOCKERHUB_USER`, `DOCKERHUB_TOKEN`) | `google-github-actions/auth` using GCP Service Account Key or Workload Identity Federation (WIF) |
| **Registry Configuration** | Docker Hub registry | Google Artifact Registry (`<region>-docker.pkg.dev/<project-id>/<repository-name>`) |
| **Image Tagging & Push** | `docker/build-push-action` pushing to `${{ secrets.DOCKERHUB_USER }}/app:${{ github.sha }}` | `docker/build-push-action` pushing to `<region>-docker.pkg.dev/<project-id>/<repository>/app:${{ github.sha }}` |
| **Deployment Stage** | `docker compose up -d` on runner | `google-github-actions/deploy-cloudrun` or `gcloud run deploy` |
| **Health Check & Verification** | `curl http://localhost:8080/health` | HTTPS endpoint verification against the Cloud Run service URL |

---

## Detailed Step-by-Step Mapping

### 1. Authentication Stage
* **Lab Approach:** Uses `docker/login-action` with static Docker Hub token credentials stored in GitHub Secrets.
* **Cloud Run Approach:** Authenticates to Google Cloud via Workload Identity Federation (keyless authentication) or a Service Account JSON key:
  ```yaml
  - name: Authenticate to Google Cloud
    uses: google-github-actions/auth@v2
    with:
      workload_identity_provider: 'projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/POOL_NAME/providers/PROVIDER_NAME'
      service_account: 'ci-deployer@PROJECT_ID.iam.gserviceaccount.com'
  ```

### 2. Registry Authentication & Push Stage
* **Lab Approach:** Pushes directly to Docker Hub under the user's namespace.
* **Cloud Run Approach:** Configures Docker to authenticate against Google Artifact Registry:
  ```yaml
  - name: Set up Docker Auth for Artifact Registry
    run: |
      gcloud auth configure-docker <region>-docker.pkg.dev --quiet

  - name: Build and Push to Artifact Registry
    uses: docker/build-push-action@v5
    with:
      context: .
      push: true
      tags: <region>-docker.pkg.dev/${{ secrets.GCP_PROJECT_ID }}/my-repo/app:${{ github.sha }}
  ```

### 3. Deploy Stage
* **Lab Approach:** Runs `docker compose up -d` on a local/runner environment.
* **Cloud Run Approach:** Deploys a managed, serverless container instance via Cloud Run:
  ```yaml
  - name: Deploy to Cloud Run
    uses: google-github-actions/deploy-cloudrun@v2
    with:
      service: app-service
      image: <region>-docker.pkg.dev/${{ secrets.GCP_PROJECT_ID }}/my-repo/app:${{ github.sha }}
      region: us-central1
  ```
