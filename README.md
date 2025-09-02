# IDM CI/CD Pipeline

This project explains a **GitLab CI/CD pipeline** to build, test, and deploy the `idm` application to multiple environments (`dev`, `test`, `stage`, and `prod`).
The pipeline has been **migrated from Helm2 to Helm3** and leverages GitLab templates for modularity.

---

## Pipeline Overview

The main pipeline includes two reusable templates from the shared DevOps repository:
https://gitlab.iponweb.net/it/iam/idm/idm/-/blob/master/.gitlab-ci.yml?ref_type=heads

- **Sonar & Maven testing** → Runs `mvn test` with SonarQube analysis.
https://gitlab.iponweb.net/devops/ci-cd/gitlab/pipelines/-/blob/master/templates/.gitlab_ci-sonar_java21_testing.yml?ref_type=heads
- **Kubernetes app build & deploy** → Builds the Docker image and deploys it with Helm3.
https://gitlab.iponweb.net/devops/ci-cd/gitlab/pipelines/-/blob/master/templates/.gitlab_ci-k8s_app_build_and_deploy.yml?ref_type=heads
---

## Stages

### 1. Test (Sonar + Maven)
- Executes unit tests and static code analysis.
- Uses the configured Sonar project key.
- Sonar errors are ignored by default (`IGNORE_SONAR_ERRORS=true`).

### 2. Build Docker Image
- Uses **Kaniko** to build the application image.
- Pushes the image to both:
  - Artifactory: `artifactory.iponweb.net/it-docker-local/...`
  - AWS ECR: `<AWS_ACCOUNT>.dkr.ecr.<region>.amazonaws.com/...`

### 3. Deploy to Dev
- Deploys automatically if `ENABLE_DEPLOY_TO_DEV=true`.
- Uses Helm3 with secrets management (`helm-secrets`).
- Installs/updates release in namespace `default`.

### 4. Deploy to Test
- Deploys automatically if `ENABLE_DEPLOY_TO_TEST=true`.
- Uses Helm3 with `helm-secrets`.
- Installs/updates release in namespace defined by `TEST_RELEASE_NS`.

### 5. Deploy to Stage
- Triggered **manually** for tags matching `*-release`.
- Schema validation is performed:
  - Compares `values.yaml` and `secrets.yaml` keys between `test` and `stage`.
  - Fails if there are mismatches.
- Deploys to namespace defined by `STAGE_RELEASE_NS`.

### 6. Deploy to Prod
- Triggered **manually** for tags matching `*-release`.
- Schema validation is performed:
  - Compares `values.yaml` and `secrets.yaml` keys between `test` and `prod`.
  - Fails if mismatches are found.
- Deploys to namespace defined by `PROD_RELEASE_NS`.

---

## Docker Image for Helm Deployments

The deployments use a custom helper image:
artifactory.iponweb.net/it-docker-local/ci/gitlab-slaves/it_k8s:x.x.x

```docker build -t it_k8s:0.1.5 .```

```docker tag it_k8s:0.1.5 artifactory.iponweb.net/it-docker-local/ci/gitlab-slaves/it_k8s:0.1.5```

```docker push artifactory.iponweb.net/it-docker-local/ci/gitlab-slaves/it_k8s:0.1.5```

### Dockerifle Tools

- **Helm 3.14.4** – Package manager for Kubernetes (replacing Helm2).
- **helm-secrets plugin** – For managing encrypted secrets with SOPS.
- **SOPS 3.7.3** – Secret management tool for encrypting YAML files.
- **yq v4.46.1 (Go version)** – YAML processor for schema checks and transformations.
- **yamldiff (Python version)** – Used to compare `values.yaml` and `secrets.yaml` schemas across environments.
- **AWS CLI** – To authenticate with AWS EKS and ECR.
- **jq** – For parsing JSON output from AWS CLI.
- **git, curl, bash, tar** – Essential utilities for scripting and automation.

## RBAC Considerations with Helm3

With **Helm2**, the Tiller component handled deployments inside the cluster and required its own RBAC permissions.
Since **Helm3 removed Tiller**, Helm now talks directly to the Kubernetes API using the user’s service account.

To ensure our CI/CD jobs can successfully deploy resources, we created a dedicated **ClusterRole** (`helm-clusterrole.yaml`) that grants Helm the necessary permissions across multiple API groups (Core, Apps, Extensions, RBAC, Networking, Monitoring).

This role allows Helm to:
- Manage Pods, ConfigMaps, Secrets, Services, Deployments, ReplicaSets, and Ingresses.
- Handle RBAC resources (`Roles`, `RoleBindings`).
- Work with Prometheus-related CRDs (`PrometheusRules`, `ServiceMonitors`).

Without these extra rights, some charts (especially those with RBAC, Ingress, or Monitoring objects) would fail to deploy under Helm3.
