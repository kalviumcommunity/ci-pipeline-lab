# Cloud Run CI/CD Pipeline Mapping Note

This document outlines how the Docker Hub + Docker Compose CI/CD pipeline maps to a production-grade deployment on **Google Cloud Run**.

---

## 1. Push Stage Mapping

| Component | Current Pipeline (Docker Hub) | Cloud Run Equivalent (Google Cloud) |
| --- | --- | --- |
| **Registry** | Docker Hub (`docker.io/<user>/app:<sha>`) | **Google Artifact Registry** (`<location>-docker.pkg.dev/<project-id>/<repo-name>/app:<sha>`) |
| **Action** | `docker/build-push-action@v5` | `docker/build-push-action@v5` targeting Artifact Registry host |

### GCP Push Stage Example
```yaml
- name: Build and Push to Artifact Registry
  uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: us-central1-docker.pkg.dev/${{ secrets.GCP_PROJECT_ID }}/my-repo/app:${{ github.sha }}
```

---

## 2. Deploy Stage Mapping

| Component | Current Pipeline (Docker Compose) | Cloud Run Equivalent (Google Cloud) |
| --- | --- | --- |
| **Runtime Target** | Local Docker Engine via `docker compose up -d` | **Google Cloud Run** fully managed container runtime |
| **Deploy Command** | `docker compose up -d` | `gcloud run deploy` or `google-github-actions/deploy-cloudrun@v2` |

### GCP Deploy Stage Example
```yaml
- name: Deploy to Cloud Run
  uses: google-github-actions/deploy-cloudrun@v2
  with:
    service: my-app-service
    region: us-central1
    image: us-central1-docker.pkg.dev/${{ secrets.GCP_PROJECT_ID }}/my-repo/app:${{ github.sha }}
```

---

## 3. Authentication Mapping

| Component | Current Pipeline (Docker Hub) | Cloud Run Equivalent (Google Cloud) |
| --- | --- | --- |
| **Credentials** | `DOCKERHUB_USER` & `DOCKERHUB_TOKEN` secrets | **GCP Service Account Key** (`GCP_SA_KEY`) or **Workload Identity Federation (WIF)** |
| **Auth Action** | `docker/login-action@v3` | `google-github-actions/auth@v2` |

### GCP Auth Example
```yaml
- name: Authenticate to Google Cloud
  uses: google-github-actions/auth@v2
  with:
    workload_identity_provider: 'projects/123456789/locations/global/workloadIdentityPools/my-pool/providers/my-provider'
    service_account: 'my-service-account@my-project.iam.gserviceaccount.com'
```

---

## Summary Flow Comparison

- **Current Pipeline**: `git push` → Auth Docker Hub → Build & Tag `user/app:sha` → Push Docker Hub → `docker compose up -d` → Health Check
- **Cloud Run Pipeline**: `git push` → Auth GCP (`google-github-actions/auth`) → Build & Tag Artifact Registry → Push AR → `gcloud run deploy` → Health Check
