## The Core Flow

Your pipeline looks like this:

**Code → GitLab CI/CD → Docker Image → Artifact Registry → Cloud Run**

Each environment (dev, staging, prod) gets its own Cloud Run service, and promotions between environments are controlled by GitLab pipelines.

---

## Step 1: Project Structure

Set up your Spring Boot project with environment-specific configs:

```
my-app/
├── src/
├── Dockerfile
├── .gitlab-ci.yml
└── pom.xml
```

**Dockerfile:**
```dockerfile
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Spring Boot reads config from environment variables at runtime, so you don't bake environment config into the image. One image, many environments.

---

## Step 2: Versioning Strategy

Use semantic versioning tied to Git. A good approach:

- **Feature branches** → `1.2.3-feature-xyz-SNAPSHOT`
- **`main` branch** → `1.2.3-SNAPSHOT` (deployed to dev automatically)
- **Git tags** (e.g., `v1.2.3`) → `1.2.3` (the release artifact)

In your `pom.xml`, keep the version as `${revision}` and pass it in via CI:

```xml
<version>${revision}</version>
```

---

## Step 3: GitLab CI/CD Pipeline

This is the heart of it. Your `.gitlab-ci.yml`:

```yaml
stages:
  - build
  - publish
  - deploy-dev
  - deploy-staging
  - deploy-prod

variables:
  IMAGE_BASE: us-docker.pkg.dev/$GCP_PROJECT_ID/my-repo/my-app

# Derive a version: use git tag if present, else branch+commit
.version_script: &version_script
  - |
    if [[ -n "$CI_COMMIT_TAG" ]]; then
      export APP_VERSION=$CI_COMMIT_TAG
    else
      export APP_VERSION="${CI_COMMIT_REF_SLUG}-${CI_COMMIT_SHORT_SHA}"
    fi
    export IMAGE_TAG="$IMAGE_BASE:$APP_VERSION"

build:
  stage: build
  image: maven:3.9-eclipse-temurin-21
  script:
    - *version_script
    - mvn package -DskipTests -Drevision=$APP_VERSION
  artifacts:
    paths:
      - target/*.jar

publish:
  stage: publish
  image: google/cloud-sdk:alpine
  services:
    - docker:dind
  script:
    - *version_script
    - echo $GCP_SA_KEY | gcloud auth activate-service-account --key-file=-
    - gcloud auth configure-docker us-docker.pkg.dev
    - docker build -t $IMAGE_TAG .
    - docker push $IMAGE_TAG
  needs: [build]

deploy-dev:
  stage: deploy-dev
  image: google/cloud-sdk:alpine
  script:
    - *version_script
    - echo $GCP_SA_KEY | gcloud auth activate-service-account --key-file=-
    - |
      gcloud run deploy my-app-dev \
        --image=$IMAGE_TAG \
        --region=us-central1 \
        --project=$GCP_PROJECT_ID \
        --set-env-vars="SPRING_PROFILES_ACTIVE=dev" \
        --allow-unauthenticated
  needs: [publish]
  only:
    - main

deploy-staging:
  stage: deploy-staging
  image: google/cloud-sdk:alpine
  script:
    - *version_script
    - |
      gcloud run deploy my-app-staging \
        --image=$IMAGE_TAG \
        --region=us-central1 \
        --project=$GCP_PROJECT_ID \
        --set-env-vars="SPRING_PROFILES_ACTIVE=staging"
  needs: [publish]
  only:
    - tags  # Only release tags trigger staging+prod
  when: on_success

deploy-prod:
  stage: deploy-prod
  image: google/cloud-sdk:alpine
  script:
    - *version_script
    - |
      gcloud run deploy my-app-prod \
        --image=$IMAGE_TAG \
        --region=us-central1 \
        --project=$GCP_PROJECT_ID \
        --set-env-vars="SPRING_PROFILES_ACTIVE=prod" \
        --no-traffic  # Deploy but send 0% traffic initially
    - |
      # Gradually shift traffic (canary)
      gcloud run services update-traffic my-app-prod \
        --to-latest --region=us-central1
  needs: [deploy-staging]
  when: manual  # Require human approval for prod
  only:
    - tags
```

---

## Step 4: Environment Config in Spring Boot

Use `application-{profile}.yml` files or inject everything via Cloud Run environment variables. The Cloud Run approach is cleaner — store secrets in **Secret Manager** and reference them:

```bash
gcloud run deploy my-app-prod \
  --set-secrets="DB_PASSWORD=db-password:latest" \
  --set-env-vars="SPRING_PROFILES_ACTIVE=prod,DB_HOST=..."
```

In Spring Boot, these become available via `@Value("${db.password}")` when you bind them to `DB_PASSWORD` in your `application.yml`:

```yaml
db:
  password: ${DB_PASSWORD}
```

---

## Step 5: Traffic Management & Rollbacks

Cloud Run makes rollbacks trivial because every deployment is a **revision**. To roll back:

```bash
# List revisions
gcloud run revisions list --service=my-app-prod --region=us-central1

# Roll back to a previous revision
gcloud run services update-traffic my-app-prod \
  --to-revisions=my-app-prod-00023-abc=100 \
  --region=us-central1
```

For a **canary release**, split traffic between revisions:

```bash
gcloud run services update-traffic my-app-prod \
  --to-revisions=my-app-prod-00024-xyz=10,my-app-prod-00023-abc=90 \
  --region=us-central1
```

---

## Step 6: The Release Workflow in Practice

Here's what day-to-day usage looks like:

1. **Developer merges to `main`** → pipeline auto-builds and deploys to dev.
2. **QA signs off on dev** → a release manager creates a Git tag (`v1.5.0`).
3. **Tag triggers pipeline** → image is built with the release version, deployed automatically to staging.
4. **Staging validation passes** → release manager clicks "play" on the manual `deploy-prod` job in GitLab.
5. **Production deploy** → Cloud Run deploys as a new revision with 0% traffic, you validate, then shift traffic.
6. **If something goes wrong** → one `gcloud` command rolls traffic back to the previous revision in seconds.

---

## GitLab Variables to Configure

In your GitLab project under **Settings → CI/CD → Variables**, add:

| Variable | Value |
|---|---|
| `GCP_PROJECT_ID` | Your GCP project ID |
| `GCP_SA_KEY` | Service account JSON key (mark as **masked**) |

The service account needs roles: `Cloud Run Admin`, `Artifact Registry Writer`, `Service Account User`.

---

## Key Takeaways

The beauty of this setup is that your **Docker image is the immutable artifact** — the same image that passes staging goes to prod, with environment behavior controlled entirely by runtime config. GitLab tags act as your release gate, and Cloud Run's revision model gives you instant rollbacks and canary deployments without any extra infrastructure.
