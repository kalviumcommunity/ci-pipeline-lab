# Cloud Run Mapping

The workflow currently deploys the commit-tagged image to the local Docker
runtime on the GitHub Actions runner. The same stages map to Google Cloud as
follows:

| Pipeline stage | Current workflow | Cloud Run equivalent |
| --- | --- | --- |
| Authenticate | `docker/login-action` with `DOCKERHUB_USER` and `DOCKERHUB_TOKEN` | `google-github-actions/auth` with a service account credential or workload identity federation |
| Push image | Docker Hub: `${DOCKERHUB_USER}/app:${GITHUB_SHA}` | Artifact Registry: `REGION-docker.pkg.dev/PROJECT/REPOSITORY/app:${GITHUB_SHA}` |
| Deploy | `docker compose pull` and `docker compose up -d --no-build` | `gcloud run deploy SERVICE --image REGION-docker.pkg.dev/PROJECT/REPOSITORY/app:${GITHUB_SHA} --region REGION --platform managed` |
| Verify | `curl -f http://localhost:8080/health` | `curl -f https://SERVICE-...run.app/health` |

The image remains tagged with the commit SHA in both versions, so each Cloud
Run revision can be traced back to the commit that produced it. This mapping
is documentation only; this assignment does not deploy to Google Cloud.