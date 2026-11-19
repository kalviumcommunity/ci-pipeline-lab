# Mapping the pipeline to Google Cloud Run

This is a reference only; it does not deploy resources.

## Pipeline changes

Push the immutable commit-tagged image to Artifact Registry:

```bash
docker push <region>-docker.pkg.dev/<project>/<repo>/app:${GITHUB_SHA}
```

Replace Docker Hub login with either `gcloud auth configure-docker <region>-docker.pkg.dev` or `google-github-actions/auth`. Prefer Workload Identity Federation for GitHub Actions over a long-lived service-account JSON key.

Deploy the published image:

```bash
gcloud run deploy app \
  --image <region>-docker.pkg.dev/<project>/<repo>/app:${GITHUB_SHA} \
  --region <region>
```

Use a dedicated service account with least-privilege roles: Artifact Registry Writer, Cloud Run Admin, and Service Account User.

| Current step | Cloud Run equivalent |
| --- | --- |
| Docker Hub login | Authenticate with Workload Identity Federation (`google-github-actions/auth`) and configure Artifact Registry with `gcloud auth configure-docker`. |
| Build and push | Build and push the SHA-tagged image to `<region>-docker.pkg.dev/<project>/<repo>/app:${GITHUB_SHA}`. |
| `docker compose up -d` | Run `gcloud run deploy app --image <region>-docker.pkg.dev/<project>/<repo>/app:${GITHUB_SHA} --region <region>`. |
