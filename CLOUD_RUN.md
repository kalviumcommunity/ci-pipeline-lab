
# Cloud Run Deployment Mapping

The current CI/CD pipeline uses Docker Hub and Docker Compose.
The same pipeline can be adapted to deploy the application
to Google Cloud Run.

## 1. Container Registry

Current:
- Docker Hub
- Image: username/app:<commit-sha>

Cloud Run equivalent:
- Google Artifact Registry
- Image: REGION-docker.pkg.dev/PROJECT_ID/REPOSITORY/app:<commit-sha>

The GitHub Actions workflow would authenticate to Google Cloud
and push the built image to Artifact Registry.

## 2. Authentication

Current:
- Docker Hub username and access token.
- GitHub repository secrets.

Cloud Run equivalent:
- Google Cloud service account.
- Workload Identity Federation is recommended for GitHub Actions.
- Grant the service account the necessary Artifact Registry
  and Cloud Run permissions.

## 3. Image Push

Current:
- docker/build-push-action pushes the image to Docker Hub.

Cloud Run equivalent:
- Authenticate Docker to Artifact Registry.
- Push the image to the Artifact Registry repository.

## 4. Deployment

Current:
- docker compose pull
- docker compose up -d

Cloud Run equivalent:

gcloud run deploy app \
  --image REGION-docker.pkg.dev/PROJECT_ID/REPOSITORY/app:COMMIT_SHA \
  --region REGION \
  --platform managed \
  --allow-unauthenticated

## 5. Deployment Flow

GitHub Push
    |
    v
GitHub Actions
    |
    v
Build Docker Image
    |
    v
Authenticate with Google Cloud
    |
    v
Push to Artifact Registry
    |
    v
Deploy using gcloud run deploy
    |
    v
Application running on Cloud Run

Note: This document describes the migration approach only.
No Google Cloud deployment is performed as part of this assignment.
