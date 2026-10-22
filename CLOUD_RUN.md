# Cloud Run Mapping Reference

| Pipeline Stage | Local Runtime (Assignment) | Google Cloud Run Mapping |
| :--- | :--- | :--- |
| **Registry** | Docker Hub | Google Cloud Artifact Registry |
| **Push Command** | `docker push user/app:sha` | `docker push region-docker.pkg.dev/project/repo/app:sha` |
| **Auth** | `docker/login-action` + Secret | `google-github-actions/auth` + Service Account Key |
| **Deploy** | `docker compose up -d` | `gcloud run deploy --image ... --region ...` |
| **Proof** | `curl -i localhost:8080/health` (HTTP 200) | `curl -i https://service-url.run.app/health` (HTTP 200) |