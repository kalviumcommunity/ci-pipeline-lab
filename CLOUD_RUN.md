# Docker Pipeline to Google Cloud Run

## Registry and image push

The local pipeline builds and pushes a Docker image to Docker Hub.

For Google Cloud Run, push the same application image to Google Artifact Registry instead:

```bash
docker build -t REGION-docker.pkg.dev/PROJECT_ID/REPOSITORY/app:COMMIT_SHA .
docker push REGION-docker.pkg.dev/PROJECT_ID/REPOSITORY/app:COMMIT_SHA
```

The Docker Hub username and access token are represented by a GCP service account and Workload Identity Federation or a service-account key.

## Authentication

The GitHub Actions Docker Hub flow uses:

- `docker/login-action@v3`
- `DOCKERHUB_USER`
- `DOCKERHUB_TOKEN`

The Google Cloud Run equivalent uses `google-github-actions/auth` and a GCP service account. The service account needs permissions to push to Artifact Registry and deploy to Cloud Run.

## Deployment

The local pipeline deploys with Docker Compose:

```bash
docker compose up -d
```

Cloud Run deploys the pushed image directly:

```bash
gcloud run deploy SERVICE_NAME \
  --image REGION-docker.pkg.dev/PROJECT_ID/REPOSITORY/app:COMMIT_SHA \
  --region REGION \
  --allow-unauthenticated
```

The Cloud Run container must listen on the application's configured port, `8080`.

## Verification

For the local Compose service:

```bash
curl -f http://localhost:8080/health
```

After deployment, replace the local address with the Cloud Run service URL:

```bash
curl -f https://SERVICE_URL/health
```

Both checks should return HTTP 200 and `{"status":"ok"}`.
