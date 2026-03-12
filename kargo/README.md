# Kargo Promotion for Frontend Service

This directory contains Kargo configurations for implementing a GitOps promotion strategy for the frontend service.

## Quick Start

Apply all Kargo resources:

```bash
kubectl apply -k .
```

## Components

1. **Stages**: Three stages defined - dev, stage, and prod
2. **Freight**: Represents the deployable frontend artifacts
3. **Warehouse**: Monitors changes in the frontend configuration
4. **Analysis Template**: Verifies the health of the frontend deployment
5. **Promotion Policy**: Configures automatic promotion from dev to stage

## Usage

### View Available Freight

```bash
kubectl get freight -n kargo-system
```

### Manually Promote to Production

```bash
kubectl kargo promote frontend-freight --stage stage --to-stage prod -n kargo-system
```

### View Promotion History

```bash
kubectl get promotions -n kargo-system
```

## Workflow

1. Changes to `env/dev/frontend` are detected by the Warehouse
2. Argo CD deploys the changes to the dev environment
3. Kargo verifies the deployment health
4. If verification passes, Kargo automatically promotes to staging
5. Production promotion requires manual approval