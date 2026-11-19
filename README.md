# LU 4.10 – Connecting CI Pipelines to Cloud Deployment

## Engineering Ticket

The CI pipeline in this repository is intended to:

1. Build a Docker image of the service.
2. Push that image to Docker Hub.
3. Deploy the image and confirm the service is reachable.

**Currently the pipeline fails.**

Your responsibility is to investigate the GitHub Actions workflow,
identify the failure points, and restore a working deployment pipeline.

The application code, the Dockerfile, and the Compose file are all
known to be correct. Do not change the application logic. Every failure
you need to fix lives in the CI/CD configuration.

---

## The Service

A minimal Node.js HTTP service.

```text
GET /health  ->  200  {"status":"ok"}
```

Build and run it locally to confirm behaviour:

```bash
docker build -t app:local .
```

In PowerShell:

```powershell
$env:IMAGE = "app:local"
docker compose up -d
curl http://localhost:8080/health
# {"status":"ok"}
```

In Bash, set `IMAGE=app:local` for the `docker compose up -d` command.

---

## Your Task

Make the pipeline in `.github/workflows/deploy.yml` succeed end to end.

A correct run must:

- Authenticate to Docker Hub before pushing.
- Push an image tagged so it is uniquely identifiable per commit.
- Only deploy after the image has actually been published.

Investigate the failing Actions logs and reason about *why* each stage
breaks before changing anything.

---

## Setup You Will Need

Configure the following repository secrets (Settings → Secrets and
variables → Actions):

| Secret            | Purpose                          |
| ----------------- | -------------------------------- |
| `DOCKERHUB_USER`  | Your Docker Hub username         |
| `DOCKERHUB_TOKEN` | A Docker Hub access token        |

---

## Evidence to Submit

1. A green GitHub Actions run.
2. The image visible in your Docker Hub repository.
3. Local verification:

   ```bash
   curl http://localhost:8080/health
   # {"status":"ok"}
   ```

4. A Pull Request containing your fixes.
5. A short video walkthrough of the working pipeline.

---

## Submission

- GitHub Pull Request
- Video submission

**Marks:** 20 &nbsp;|&nbsp; **Time:** ~20–30 minutes
