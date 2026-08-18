# Cloud Run Deployment Mapping

This document maps the local CI/CD pipeline to Google Cloud Run for production deployment.

## Architecture Mapping

### Current Local Pipeline
```
Git Push → GitHub Actions → Docker Hub → Docker Compose (localhost)
```

### Cloud Run Pipeline
```
Git Push → GitHub Actions → Artifact Registry → Cloud Run → Load Balancer
```

## Step-by-Step Mapping

### 1. **Build Step** (Unchanged)
- **Local:** `docker/build-push-action` builds and pushes to Docker Hub
- **Cloud Run:** Same build process, but push to **Google Artifact Registry** instead
  ```yaml
  - name: Authenticate to Google Cloud
    uses: google-github-actions/auth@v2
    with:
      credentials_json: ${{ secrets.GCP_SA_KEY }}
  
  - name: Set up Cloud SDK
    uses: google-github-actions/setup-gcloud@v2
  
  - name: Configure Docker for Artifact Registry
    run: gcloud auth configure-docker us-central1-docker.pkg.dev
  
  - name: Build and push image to Artifact Registry
    uses: docker/build-push-action@v5
    with:
      context: .
      push: true
      tags: us-central1-docker.pkg.dev/${{ secrets.GCP_PROJECT_ID }}/app/app:${{ github.sha }}
  ```

### 2. **Authentication**
- **Local:** Docker Hub credentials via `DOCKERHUB_USER` and `DOCKERHUB_TOKEN` secrets
- **Cloud Run:** Google Cloud service account key (`GCP_SA_KEY`) with roles:
  - `roles/artifactregistry.writer` (push images)
  - `roles/run.admin` (deploy Cloud Run services)
  - `roles/iam.serviceAccountUser` (service account impersonation)

### 3. **Image Registry**
- **Local:** Docker Hub (`docker.io/username/app`)
- **Cloud Run:** Artifact Registry (`us-central1-docker.pkg.dev/project-id/repository/app`)

### 4. **Deployment**
- **Local:** `docker compose up -d` with environment variable `IMAGE`
- **Cloud Run:** Use `gcloud run deploy` command
  ```yaml
  - name: Deploy to Cloud Run
    run: |
      gcloud run deploy app \
        --image us-central1-docker.pkg.dev/${{ secrets.GCP_PROJECT_ID }}/app/app:${{ github.sha }} \
        --region us-central1 \
        --platform managed \
        --allow-unauthenticated \
        --port 8080
  ```

### 5. **Health Check & Verification**
- **Local:** `curl -f http://localhost:8080/health`
- **Cloud Run:** `curl -f https://<service-url>.run.app/health`

## Required GCP Setup

1. **Create a Google Cloud Project** (if not already done)
2. **Enable required APIs:**
   - Artifact Registry API
   - Cloud Run API
   - IAM API

3. **Create a service account:**
   ```bash
   gcloud iam service-accounts create github-actions \
     --display-name="GitHub Actions deployment account"
   ```

4. **Grant IAM roles:**
   ```bash
   PROJECT_ID=$(gcloud config get-value project)
   
   gcloud projects add-iam-policy-binding $PROJECT_ID \
     --member=serviceAccount:github-actions@${PROJECT_ID}.iam.gserviceaccount.com \
     --role=roles/artifactregistry.writer
   
   gcloud projects add-iam-policy-binding $PROJECT_ID \
     --member=serviceAccount:github-actions@${PROJECT_ID}.iam.gserviceaccount.com \
     --role=roles/run.admin
   
   gcloud projects add-iam-policy-binding $PROJECT_ID \
     --member=serviceAccount:github-actions@${PROJECT_ID}.iam.gserviceaccount.com \
     --role=roles/iam.serviceAccountUser
   ```

5. **Create and export the service account key:**
   ```bash
   gcloud iam service-accounts keys create key.json \
     --iam-account=github-actions@${PROJECT_ID}.iam.gserviceaccount.com
   ```

6. **Add GitHub Secrets:**
   - `GCP_PROJECT_ID`: Your Google Cloud project ID
   - `GCP_SA_KEY`: Contents of `key.json` (base64 encoded or JSON)

## Configuration Differences

| Aspect | Local | Cloud Run |
|--------|-------|-----------|
| **Image Registry** | Docker Hub | Artifact Registry |
| **Auth Method** | Docker login | GCP service account |
| **Deployment** | docker-compose | gcloud run deploy |
| **Port** | 8080 (localhost) | 8080 (managed by Cloud Run) |
| **Health Check** | curl localhost | curl service URL |
| **Scaling** | Manual | Auto-scaling by Cloud Run |
| **Cost** | Free tier limited | Pay-per-use (generous free tier) |

## Migration Path

To migrate from this local pipeline to Cloud Run:

1. Create a GCP project and service account
2. Add GCP secrets to GitHub
3. Create a new job in the workflow (e.g., `deploy-cloud`)
4. Make it conditional on a tag or branch (e.g., `if: github.ref == 'refs/heads/main'`)
5. Use the Artifact Registry and Cloud Run commands above
6. Test with a dry-run first: `gcloud run deploy --dry-run`

## Testing Cloud Run Locally

To simulate Cloud Run environment locally:

```bash
docker run -p 8080:8080 \
  -e PORT=8080 \
  us-central1-docker.pkg.dev/PROJECT_ID/app/app:latest
```

Then verify:
```bash
curl http://localhost:8080/health
```

---

**Note:** This is documentation only. The deployment workflow is configured for Docker Hub and local docker-compose. To enable Cloud Run deployment, implement the GCP authentication and deployment steps above in the GitHub Actions workflow.
