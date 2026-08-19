# Google Cloud Run Deployment Mapping Note

This document outlines how the GitHub Actions CI/CD pipeline (`.github/workflows/deploy.yml`) maps to a production Google Cloud Run deployment architecture on GCP.

---

## Architecture Mapping Summary

| Stage / Component | Current Pipeline (Docker Hub + Local Runtime) | Google Cloud Production Architecture |
| :--- | :--- | :--- |
| **Container Registry** | Docker Hub (`docker.io/<username>/app:<sha>`) | **Google Artifact Registry** (`<region>-docker.pkg.dev/<project_id>/<repo_name>/app:<sha>`) |
| **Authentication** | Docker Hub PAT (`docker/login-action`) | **GCP Service Account / WIF** (`google-github-actions/auth`) |
| **Deployment Engine** | Local Docker Compose (`docker compose up -d`) | **Google Cloud Run** (`gcloud run deploy`) |

---

## 1. Registry Mapping (Push Stage)

- **Local Pipeline:** Container image is pushed to Docker Hub:
  ```yaml
  - name: Build and push image
    uses: docker/build-push-action@v5
    with:
      context: .
      push: true
      tags: ${{ secrets.DOCKERHUB_USER }}/app:${{ github.sha }}
  ```
- **Cloud Run Architecture:** Container image is pushed to Google Artifact Registry:
  ```yaml
  - name: Build and push to Artifact Registry
    uses: docker/build-push-action@v5
    with:
      context: .
      push: true
      tags: us-central1-docker.pkg.dev/${{ secrets.GCP_PROJECT_ID }}/my-repo/app:${{ github.sha }}
  ```

---

## 2. Deployment Mapping (Deploy Stage)

- **Local Pipeline:** Deployed locally using Docker Compose:
  ```yaml
  - name: Deploy with Docker Compose
    env:
      IMAGE: ${{ secrets.DOCKERHUB_USER }}/app:${{ github.sha }}
    run: docker compose up -d
  ```
- **Cloud Run Architecture:** Deployed to serverless Cloud Run via `gcloud` CLI or `google-github-actions/deploy-cloudrun`:
  ```yaml
  - name: Deploy to Google Cloud Run
    uses: google-github-actions/deploy-cloudrun@v2
    with:
      service: my-app-service
      image: us-central1-docker.pkg.dev/${{ secrets.GCP_PROJECT_ID }}/my-repo/app:${{ github.sha }}
      region: us-central1
      flags: '--allow-unauthenticated --port=8080'
  ```

---

## 3. Authentication Mapping

- **Local Pipeline:** Uses Docker Hub username and access token:
  ```yaml
  - name: Log in to Docker Hub
    uses: docker/login-action@v3
    with:
      username: ${{ secrets.DOCKERHUB_USER }}
      password: ${{ secrets.DOCKERHUB_TOKEN }}
  ```
- **Cloud Run Architecture:** Authenticates via GCP Service Account Key or Workload Identity Federation (WIF) using `google-github-actions/auth`:
  ```yaml
  - name: Authenticate to Google Cloud
    uses: google-github-actions/auth@v2
    with:
      workload_identity_provider: ${{ secrets.GCP_WIF_PROVIDER }}
      service_account: ${{ secrets.GCP_SA_EMAIL }}
  ```
  And authenticates Docker Buildx to Artifact Registry:
  ```yaml
  - name: Log in to Artifact Registry
    uses: docker/login-action@v3
    with:
      registry: us-central1-docker.pkg.dev
      username: oauth2accesstoken
      password: ${{ steps.auth.outputs.access_token }}
  ```

---

## Key Benefits of Cloud Run Mapping
1. **Immutable Artifact Traceability:** Each GitHub commit SHA (`${{ github.sha }}`) triggers a build resulting in a uniquely tagged container image in Artifact Registry.
2. **Serverless Scalability:** Cloud Run automatically scales container instances from 0 to N based on traffic.
3. **Zero-Trust Security:** Workload Identity Federation eliminates the need for long-lived static service account keys in repository secrets.
