# Docker Hub to Cloud Run Mapping

## Push

Push the built container image to Google Artifact Registry instead of Docker Hub:

```text
<region>-docker.pkg.dev/<project-id>/<repository>/<service>:<commit-sha>
```

The immutable commit SHA tag should continue to identify each release.

## Authentication

Configure a Google Cloud service account and grant it the Artifact Registry writer role for pushes. In GitHub Actions, use Workload Identity Federation or an appropriate service-account key secret. The service account should not be committed to the repository.

Cloud Run should run with a service account that has permission to pull the image. For private Artifact Registry repositories, grant that runtime service account the Artifact Registry reader role.

## Deploy

Use a mapped Cloud Run deployment command instead of local Docker Compose:

```bash
gcloud run deploy <service-name> \
  --image=<region>-docker.pkg.dev/<project-id>/<repository>/<service>:<commit-sha> \
  --region=<region> \
  --platform=managed \
  --allow-unauthenticated
```

This mapping is documentation only; this repository does not deploy to Cloud Run.
