# Cloud Run Mapping

The current pipeline deploys locally using Docker Compose.

For a future Cloud Run deployment:

| Local Pipeline | Cloud Run |
|---|---|
| Docker Hub | Artifact Registry |
| docker push | docker push REGION-docker.pkg.dev/... |
| Docker Hub authentication | Google service account authentication |
| docker compose up -d | gcloud run deploy |
| localhost:8080/health | Cloud Run service URL |

The pipeline flow remains:

Build → Tag → Authenticate → Push → Deploy

Only the registry and deployment target change.