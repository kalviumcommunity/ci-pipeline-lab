# Cloud Run Mapping

This pipeline uses Docker Hub and Docker Compose for the local deployment target. The same stages map directly to Cloud Run:

- Registry: replace Docker Hub with Artifact Registry, for example `REGION-docker.pkg.dev/PROJECT/REPOSITORY/ci-pipeline-lab:${{ github.sha }}`.
- Authentication: replace `docker/login-action` with `google-github-actions/auth` using a GitHub secret that stores a service account credential or Workload Identity Federation settings.
- Push: keep the immutable commit SHA tag and push the image to Artifact Registry.
- Deploy: replace `docker compose up -d --no-build` with `gcloud run deploy ci-pipeline-lab --image REGION-docker.pkg.dev/PROJECT/REPOSITORY/ci-pipeline-lab:${{ github.sha }} --region REGION --platform managed`.
- Proof: replace the local `curl http://localhost:8080/health` check with a `curl` against the Cloud Run service URL.

Rollback would use Cloud Run revision traffic controls, for example `gcloud run services update-traffic ci-pipeline-lab --to-revisions REVISION=100`.
