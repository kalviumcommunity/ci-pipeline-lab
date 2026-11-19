# Moving the deployment to Google Cloud Run

The current workflow publishes `${DOCKERHUB_USER}/app:<commit-sha>` to Docker
Hub and starts it with Docker Compose on the GitHub Actions runner. To deploy
persistently, publish the image to Google Artifact Registry (GAR) and replace
the Compose step with a Cloud Run deployment.

## 1. Configure Google Cloud

Create or select a Google Cloud project, enable billing, and enable the
Artifact Registry and Cloud Run APIs. Create a Docker-format GAR repository in
your chosen region. Create a service account with permission to write to that
repository (`roles/artifactregistry.writer`) and deploy Cloud Run services
(`roles/run.admin`); grant it `roles/iam.serviceAccountUser` on the runtime
service account used by Cloud Run.

For GitHub Actions, configure Workload Identity Federation (WIF) for your
GitHub repository and allow its provider to impersonate the deploy service
account. WIF uses GitHub's short-lived OIDC token, so you do not need to save
a service-account JSON key. Restrict the provider condition to your repository
and, if appropriate, the `main` branch.

## 2. Add GitHub configuration

In **Settings → Secrets and variables → Actions**, add these repository
variables (or use secrets if your organization requires them):

| Name | Example | Purpose |
| --- | --- | --- |
| `GCP_PROJECT_ID` | `my-project-id` | Google Cloud project |
| `GCP_REGION` | `us-central1` | GAR and Cloud Run region |
| `GAR_REPOSITORY` | `apps` | GAR Docker repository name |
| `CLOUD_RUN_SERVICE` | `my-app` | Cloud Run service name |
| `WIF_PROVIDER` | `projects/123456789/locations/global/workloadIdentityPools/github/providers/actions` | Full WIF provider resource name |
| `WIF_SERVICE_ACCOUNT` | `github-deploy@my-project-id.iam.gserviceaccount.com` | Deploy service-account email |

The existing `DOCKERHUB_USER` and `DOCKERHUB_TOKEN` secrets are only needed for
the Docker Hub workflow; the Cloud Run workflow authenticates to Google Cloud
with WIF instead.

## 3. Replace the image publish and deploy steps

Give the workflow `id-token: write` permission, then authenticate and configure
Docker for GAR. The image name must include the region, project, repository,
and commit tag:

```yaml
permissions:
  contents: read
  id-token: write

steps:
  - uses: actions/checkout@v4
  - uses: google-github-actions/auth@v2
    with:
      workload_identity_provider: ${{ vars.WIF_PROVIDER }}
      service_account: ${{ vars.WIF_SERVICE_ACCOUNT }}
  - uses: google-github-actions/setup-gcloud@v2
  - name: Configure Docker for Artifact Registry
    run: gcloud auth configure-docker "${{ vars.GCP_REGION }}-docker.pkg.dev" --quiet
  - name: Build and push image
    env:
      IMAGE: ${{ vars.GCP_REGION }}-docker.pkg.dev/${{ vars.GCP_PROJECT_ID }}/${{ vars.GAR_REPOSITORY }}/app:${{ github.sha }}
    run: |
      docker build --tag "$IMAGE" .
      docker push "$IMAGE"
  - name: Deploy to Cloud Run
    env:
      IMAGE: ${{ vars.GCP_REGION }}-docker.pkg.dev/${{ vars.GCP_PROJECT_ID }}/${{ vars.GAR_REPOSITORY }}/app:${{ github.sha }}
    run: |
      gcloud run deploy "${{ vars.CLOUD_RUN_SERVICE }}" \
        --image "$IMAGE" \
        --project "${{ vars.GCP_PROJECT_ID }}" \
        --region "${{ vars.GCP_REGION }}" \
        --platform managed \
        --allow-unauthenticated
```

In a complete workflow, keep image publishing and Cloud Run deployment in
separate jobs and make the deploy job depend on the successful publish job
with `needs:`.

## 4. Port and environment

This app reads the port from `process.env.PORT` and defaults to `8080`. Cloud
Run supplies `PORT` at runtime, so the existing Dockerfile and application can
be used without Compose's `8080:8080` port mapping. Configure any application
environment variables with Cloud Run's `--set-env-vars KEY=VALUE` option (or
Cloud Run's console); do not put credentials in the workflow or image. This
service currently needs no additional application variables. Cloud Run's
default ingress requires authentication; use `--allow-unauthenticated` only
when the health endpoint and service should be publicly reachable.
