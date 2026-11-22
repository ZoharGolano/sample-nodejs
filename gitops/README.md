# GitOps Deployment with Argo CD

## Overview

This directory contains the declarative Argo CD configuration used to deploy the sample Node.js application to the Targil2026 K3s cluster.

Argo CD implements GitOps by continuously comparing the desired application configuration stored in Git with the actual resources running in Kubernetes.

## Architecture

The deployment follows this workflow:

```text
GitHub Repository
ZoharGolano/sample-nodejs
        |
        | main branch
        v
Helm Chart
helm/sample-nodejs
        |
        v
Argo CD
        |
        | Automated Sync
        v
K3s Cluster
        |
        v
Namespace: sample-app
        |
        +-- Deployment (2 replicas)
        +-- ClusterIP Service
        +-- Traefik Ingress
```

## Repository Strategy

The application source code, CI/CD workflow, Helm chart, and GitOps configuration are maintained in the same Git repository.

This approach was selected for the exercise because it simplifies version control, change reviews, and traceability between application code and deployment configuration.

A separate infrastructure repository is used for the underlying K3s cluster and Argo CD installation.

## Argo CD Application

The Application manifest is stored in:

`gitops/application.yaml`

Configuration:

| Parameter | Value |
|---|---|
| Application name | sample-nodejs |
| Argo CD namespace | argocd |
| Repository | https://github.com/ZoharGolano/sample-nodejs.git |
| Branch | main |
| Helm chart path | helm/sample-nodejs |
| Helm release name | sample-nodejs |
| Destination namespace | sample-app |
| Automated synchronization | Enabled |
| Self-healing | Enabled |
| Pruning | Enabled |

## Deployment Process

### 1. Prepare the Application

The Application manifest is maintained in Git and reviewed through a pull request.

After merging the changes into `main`, bootstrap the Application from the Ubuntu host:

```bash
kubectl apply -f gitops/application.yaml
```

This initial bootstrap operation registers the declarative Application with Argo CD.

Subsequent application changes are managed through GitOps.

### 2. Verify Synchronization

Check the Application:

```bash
kubectl get applications -n argocd
```

Check the deployed resources:

```bash
kubectl get deployments,pods,services,ingress -n sample-app
```

The expected Application status is:

- Sync status: Synced
- Health status: Healthy

### 3. Test Application Endpoints

The application exposes the following HTTP endpoints:

| Endpoint | Purpose |
|---|---|
| /my-app | Application response |
| /ready | Kubernetes readiness probe |
| /live | Kubernetes liveness probe |
| /metrics | Prometheus metrics |

The Helm chart configures an Ingress with the host:

`sample-nodejs.test`

The hostname must resolve to the address used to reach the Traefik ingress controller.

For local testing, a custom HTTP Host header can be used instead of configuring DNS.

## Continuous Delivery

GitHub Actions performs:

1. Application tests.
2. Dependency vulnerability audit.
3. SAST scanning using Semgrep.
4. Docker image build.
5. Image vulnerability scanning using Trivy.
6. Publication of approved images to GHCR.

Images with HIGH or CRITICAL vulnerabilities are blocked by the security scan.

The Helm chart references an explicit image tag.

Argo CD detects changes to the Helm chart configuration in the `main` branch and synchronizes the desired state with Kubernetes.

Image promotion currently requires updating the image tag in Git; automated image promotion is not yet configured.

## Security

- Application manifests are version-controlled.
- Deployment changes are reviewed through pull requests.
- Container images are scanned before publication.
- The application container runs as a non-root user.
- Resource requests and limits are configured.
- Readiness and liveness probes are enabled.
- GitOps synchronization detects and corrects configuration drift.

## Troubleshooting

Inspect the Argo CD Application:

```bash
kubectl describe application sample-nodejs -n argocd
```

Check application events:

```bash
kubectl get events -n sample-app --sort-by=.lastTimestamp
```

Inspect application logs:

```bash
kubectl logs -n sample-app deployment/sample-nodejs
```

Inspect the Argo CD repository server:

```bash
kubectl logs -n argocd deployment/argocd-repo-server
```

## Expected Outcome

After successful synchronization, Argo CD should report the Application as Healthy and Synced.

The Node.js application should run with two replicas and be accessible through the Traefik ingress controller.
