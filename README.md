# ElderPing Shared

## Overview

ElderPing Shared is a repository containing reusable GitHub Actions workflows and shared configurations for the ElderPing platform. It provides standardized CI/CD pipelines, security scanning, and deployment workflows that are used across all ElderPing microservices.

## Purpose

The shared repository enables:
- **DRY Principle**: Don't Repeat Yourself - common workflows defined once
- **Consistency**: All services use the same CI/CD standards
- **Maintainability**: Update workflows in one place, affects all services
- **Standardization**: Uniform security scanning and deployment processes

## Structure

```
elderping-shared/
└── .github/
    └── workflows/
        ├── ci-docker-publish.yml      # Docker build and publish workflow
        ├── ci-sast.yml                 # Static Application Security Testing
        ├── ci-sca.yml                  # Software Composition Analysis
        ├── ci-trivy.yml                # Trivy vulnerability scanning
        └── cd-template.yml             # GitOps deployment template
```

## Workflows

### 1. ci-docker-publish.yml

Reusable workflow for building and publishing Docker images.

**Features:**
- Multi-architecture Docker builds (linux/amd64, linux/arm64)
- Docker Hub authentication
- SHA-based tagging for develop branch
- Release tag-based tagging for main branch
- Image existence check before building

**Inputs:**
- `service-name`: Name of the service
- `image-repo`: Docker Hub repository
- `environment`: Target environment (dev/prod)

**Usage:**
```yaml
uses: ElderPing/elderping-shared/.github/workflows/ci-docker-publish.yml@main
with:
  service-name: auth-service
  image-repo: arunnsimon/elderpinq-auth-service
  environment: dev
```

### 2. ci-sast.yml

Static Application Security Testing workflow.

**Features:**
- CodeQL analysis
- Security vulnerability scanning
- SARIF report generation
- GitHub Security tab integration

**Usage:**
```yaml
uses: ElderPing/elderping-shared/.github/workflows/ci-sast.yml@main
```

### 3. ci-sca.yml

Software Composition Analysis workflow.

**Features:**
- Dependency scanning
- Vulnerability detection in dependencies
- License compliance checking
- Dependency update recommendations

**Usage:**
```yaml
uses: ElderPing/elderping-shared/.github/workflows/ci-sca.yml@main
```

### 4. ci-trivy.yml

Trivy vulnerability scanning workflow.

**Features:**
- Container image scanning
- File system scanning
- Configuration scanning
- CVE database checking
- Severity-based fail thresholds

**Usage:**
```yaml
uses: ElderPing/elderping-shared/.github/workflows/ci-trivy.yml@main
with:
  image-repo: arunnsimon/elderpinq-auth-service
  image-tag: dev-latest
```

### 5. cd-template.yml

GitOps deployment template for updating Helm charts.

**Features:**
- Clones Helm charts repository
- Updates image tag in values files using yq
- Validates YAML syntax
- Commits and pushes changes
- ArgoCD auto-sync integration

**Inputs:**
- `service-name`: Name of the service folder in charts repo
- `image-tag`: New image tag to update
- `environment`: Target environment (main/develop)
- `target-env`: values-dev.yaml or values-prod.yaml
- `charts-repo`: Helm charts repository (default: ElderPing/elderping-k8s-charts)
- `charts-path`: Path prefix in charts repo (default: microservices)

**Secrets:**
- `HELM_REPO_PAT`: PAT with push access to Helm charts repository

**Usage:**
```yaml
uses: ElderPing/elderping-shared/.github/workflows/cd-template.yml@main
with:
  service-name: auth-service
  image-tag: dev-latest
  environment: develop
  target-env: dev
secrets:
  HELM_REPO_PAT: ${{ secrets.HELM_REPO_PAT }}
```

## Service-Specific Workflows

Each ElderPing service has its own CI workflow that calls the shared workflows:

**Example: elderping-auth-service/.github/workflows/ci-auth-service.yaml**

```yaml
name: CI - Auth Service

on:
  push:
    branches: [develop, main]
  pull_request:
    branches: [develop, main]

jobs:
  security:
    uses: ElderPing/elderping-shared/.github/workflows/ci-sast.yml@main
  
  sca:
    uses: ElderPing/elderping-shared/.github/workflows/ci-sca.yml@main
  
  build:
    uses: ElderPing/elderping-shared/.github/workflows/ci-docker-publish.yml@main
    with:
      service-name: auth-service
      image-repo: arunnsimon/elderpinq-auth-service
      environment: ${{ github.ref == 'refs/heads/main' && 'prod' || 'dev' }}
  
  deploy:
    if: github.ref == 'refs/heads/develop' || github.ref == 'refs/heads/main'
    uses: ElderPing/elderping-shared/.github/workflows/cd-template.yml@main
    with:
      service-name: auth-service
      image-tag: ${{ github.ref == 'refs/heads/main' && 'prod-latest' || 'dev-latest' }}
      environment: ${{ github.ref == 'refs/heads/main' && 'main' : 'develop' }}
      target-env: ${{ github.ref == 'refs/heads/main' && 'prod' : 'dev' }}
    secrets:
      HELM_REPO_PAT: ${{ secrets.HELM_REPO_PAT }}
```

## Deployment Strategy

### Development Branch (develop)
1. Code pushed to develop branch
2. Security scans run (SAST, SCA, Trivy)
3. Docker image built and tagged as `dev-latest`
4. Image pushed to Docker Hub
5. Helm chart values-dev.yaml updated with `dev-latest` tag
6. ArgoCD syncs to elderping-dev namespace

### Production Branch (main)
1. Release created on main branch
2. Security scans run (SAST, SCA, Trivy)
3. Docker image built and tagged as `prod-latest` (or release tag)
4. Image pushed to Docker Hub
5. Helm chart values-prod.yaml updated with `prod-latest` tag
6. ArgoCD syncs to elderping-prod namespace

## Required Secrets

### For All Services
- `HELM_REPO_PAT`: GitHub Personal Access Token with push access to elderping-k8s-charts

### For Docker Publishing
- `DOCKER_USERNAME`: Docker Hub username
- `DOCKER_PASSWORD`: Docker Hub password or access token

## Configuration

### Service Name Conversion

The `cd-template.yml` workflow automatically converts kebab-case service names to camelCase for Helm chart values:

- `auth-service` → `authService`
- `health-service` → `healthService`
- `reminder-service` → `reminderService`
- `alert-service` → `alertService`
- `ui-service` → `uiService`

This is done using sed in the workflow:
```bash
SERVICE_NAME_CAMEL=$(echo "${{ inputs.service-name }}" | sed 's/-\([a-z]\)/\U\1/g')
```

## Maintenance

### Updating Workflows

To update a shared workflow:
1. Make changes to the workflow file in elderping-shared
2. Commit and push to main branch
3. All services using the workflow will automatically use the updated version on their next run

### Versioning

Workflows are referenced by branch or tag:
- `@main` - Always use latest version (recommended for rapid iteration)
- `@v1.0.0` - Use specific version (recommended for stability)

## Troubleshooting

### Common Issues

**Workflow Not Found**
- Verify the workflow path is correct
- Check the branch/tag reference
- Ensure elderping-shared repository is accessible

**HELM_REPO_PAT Permission Denied**
- Verify PAT has write access to elderping-k8s-charts
- Check PAT has not expired
- Ensure PAT is added as a secret in the service repository

**Image Tag Update Failed**
- Verify yq is installed in the runner
- Check the values file path is correct
- Ensure service-name matches the folder name in charts repo

**Docker Build Failed**
- Verify Docker Hub credentials are correct
- Check Dockerfile exists in the service repository
- Ensure build context is correct

## Contributing

1. Create feature branch from main
2. Make changes to workflows
3. Test changes in a service repository
4. Commit with descriptive message
5. Create pull request to main
6. Update service repositories to use new version if needed

## License

Proprietary - ElderPing Platform
