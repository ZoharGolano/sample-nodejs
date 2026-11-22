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


## Deployment Verification — November 22, 2026

The sample Node.js application was successfully deployed to the K3s cluster using Argo CD.

### Argo CD Status

Command:

```bash
kubectl get applications -n argocd
```

Observed result:

```text
NAME            SYNC STATUS   HEALTH STATUS
sample-nodejs   Synced        Healthy
```

### Kubernetes Deployment

Command:

```bash
kubectl get deployments,pods,svc,ingress -n sample-app
```

Observed results:

- Deployment: `sample-nodejs`
- Desired replicas: 2
- Available replicas: 2
- Pods: 2 running, 0 restarts
- Service: `sample-nodejs`, ClusterIP, port 80
- Ingress controller: Traefik
- Ingress hostname: `sample-nodejs.test`
- Ingress addresses: `192.168.122.11`, `192.168.122.12`, `192.168.122.13`

### HTTP Endpoint Validation

The tests were executed from the Ubuntu host against the Traefik ingress controller.

**Application endpoint:**

```bash
curl -i -H "Host: sample-nodejs.test" \
  http://192.168.122.11/my-app
```

Observed status: `HTTP/1.1 200 OK`.

**Readiness endpoint:**

```bash
curl -i -H "Host: sample-nodejs.test" \
  http://192.168.122.11/ready
```

Observed status: `HTTP/1.1 200 OK`.

Observed response: `Ready`.

**Liveness endpoint:**

```bash
curl -i -H "Host: sample-nodejs.test" \
  http://192.168.122.11/live
```

Observed status: `HTTP/1.1 200 OK`.

**Prometheus metrics endpoint:**

```bash
curl -s -H "Host: sample-nodejs.test" \
  http://192.168.122.11/metrics | head -n 15
```

The endpoint returned Prometheus-formatted process metrics, including:

- `process_cpu_user_seconds_total`
- `process_cpu_system_seconds_total`
- `process_cpu_seconds_total`
- `process_start_time_seconds`

### Verification Result

Argo CD reported the application as `Synced` and `Healthy`.

The Deployment had two available replicas, and HTTP requests through Traefik returned successful responses.

The `/metrics` endpoint was accessible and returned Prometheus-formatted metrics.

**Note:** Prometheus scraping and automated reconciliation after a Git change have not yet been tested. These are separate verification steps.


## Troubleshooting: Duplicate Helm Values

### Issue

During GitOps verification on November 22, 2026, duplicate top-level keys were discovered in `helm/sample-nodejs/values.yaml`.

The file contained the original values generated by `helm create`, followed by the application's custom configuration.

For example, the file included two definitions of `replicaCount`:

```yaml
replicaCount: 1
```

and later:

```yaml
replicaCount: 2
```

The same issue affected other configuration sections, including `image`, `service`, `ingress`, and `resources`.

### Observed Behavior

Despite the duplicate definitions, the Kubernetes Deployment was running with two replicas and the expected GHCR container image.

The Helm rendering process used the later values in this environment.

However, duplicate YAML mapping keys are ambiguous and should not be relied upon.

### Resolution

The original, unused values section was removed, leaving a single definition for each application configuration key.

The cleanup removed 119 lines from `values.yaml`.

### Validation

The following commands were used:

```bash
helm lint helm/sample-nodejs
```

Result:

```text
1 chart(s) linted, 0 chart(s) failed
```

Render the Helm manifests:

```bash
helm template sample-nodejs helm/sample-nodejs \
  --namespace sample-app \
  > /tmp/sample-nodejs-clean-rendered.yaml
```

Verify the configuration:

```bash
grep -E 'replicas:|image:' \
  /tmp/sample-nodejs-clean-rendered.yaml
```

The rendered Deployment retained:

- Two replicas.
- The expected GHCR image.
- Non-root execution.
- Disabled privilege escalation.
- CPU and memory requests and limits.

### Lesson Learned

Helm charts should contain a single definition for each YAML mapping key.

Always validate rendered Kubernetes manifests, not only the original Helm values.

A successful deployment does not necessarily mean the source configuration is clean or maintainable.



## Argo CD Automated Synchronization Verification

### Objective

Verify that Argo CD automatically reconciles changes made to the Helm chart in GitHub without requiring manual Kubernetes deployment commands.

### Test Environment

- Kubernetes: K3s v1.37.2+k3s1
- Argo CD: v3.6.2
- Application: `sample-nodejs`
- Namespace: `sample-app`
- Git repository: `ZoharGolano/sample-nodejs`
- Target branch: `main`

### Test 1: Scale Up

The Helm configuration was updated through a GitHub pull request:

```yaml
replicaCount: 3
```

After merging the pull request, Argo CD detected and synchronized the change.

Observed results:

- Argo CD Sync Status: `Synced`
- Argo CD Health Status: `Healthy`
- Deployment: `3/3` replicas available
- Pods: 3 running
- Container restarts: 0

The two existing Pods remained running, and Kubernetes created a third Pod.

**Result: PASSED**

### Test 2: Scale Down

The Helm configuration was restored through another GitHub pull request:

```yaml
replicaCount: 2
```

After merging the pull request, Argo CD automatically reconciled the Deployment.

Observed results:

- Argo CD Sync Status: `Synced`
- Argo CD Health Status: `Healthy`
- Deployment: `2/2` replicas available
- Pods: 2 running
- Container restarts: 0

The additional Pod was removed while the original two Pods remained running.

**Result: PASSED**

### Verification Commands

```bash
kubectl get applications -n argocd

kubectl get deployment sample-nodejs -n sample-app

kubectl get pods -n sample-app
```

### Conclusion

Both scaling tests completed successfully.

Changes to the Helm chart were applied through GitHub pull requests, and Argo CD automatically synchronized the desired state with the Kubernetes cluster.

No manual `kubectl scale`, `kubectl apply`, or `helm upgrade` commands were required during either scaling operation.

The application is currently running with two replicas and reports `Synced` and `Healthy` in Argo CD.



## Application Versioning and Image Promotion

### Versioning Strategy

The project uses Semantic Versioning (MAJOR.MINOR.PATCH) for the Node.js application.

The Helm chart is versioned independently.

Application version information is maintained in:

- `package.json` and `package-lock.json` — application version.
- `helm/sample-nodejs/Chart.yaml` — Helm chart version and application version.
- `helm/sample-nodejs/values.yaml` — exact container image deployed by Argo CD.

### Version Bump Procedure

For a patch release, update the application version:

```bash
npm version patch --no-git-tag-version
```

When Node.js is not installed on the host, run npm using Docker:

```bash
docker run --rm --network host \
  --user "$(id -u):$(id -g)" \
  -v "$PWD:/app" -w /app \
  node:22-alpine \
  npm version patch --no-git-tag-version
```

Update the Helm chart metadata to match the release:

```yaml
version: 0.2.1
appVersion: "1.0.1"
```

Validate the Helm chart:

```bash
helm lint helm/sample-nodejs
```

### CI/CD and Image Publication

After merging the version bump into `main`, GitHub Actions runs application tests, dependency auditing, SAST, Docker image building, and Trivy scanning.

A container image is published to GHCR only when the required checks succeed.

The published image receives a commit-based tag:

```text
sha-<first-12-characters-of-commit-SHA>
```

### Image Promotion Through GitOps

Image publication and deployment are separate operations.

The Helm chart uses an explicit image tag, ensuring that deployments reference a specific approved build.

After verifying the new image is available in GHCR, update `helm/sample-nodejs/values.yaml` with the published image tag through a pull request.

Argo CD automatically synchronizes the updated Helm configuration after the pull request is merged into `main`.

This provides a reviewable deployment history and allows previous image versions to be restored through Git.

### Version 1.0.1 Release

The first versioning exercise updates:

| Component | Previous | New |
|---|---|---|
| Application | 1.0.0 | 1.0.1 |
| Helm chart | 0.2.0 | 0.2.1 |
| Helm appVersion | 1.0.0 | 1.0.1 |

The container image will be promoted separately after successful CI/CD validation.

This approach provides controlled image promotion rather than automatically deploying every newly published image.



## Version 1.0.1 — Release and GitOps Promotion Verification

### Objective

Verify the complete application release process, from version bumping and security validation to image publication and deployment through Argo CD.

### Release Information

| Component | Value |
|---|---|
| Application version | 1.0.1 |
| Helm chart version | 0.2.1 |
| Helm appVersion | 1.0.1 |
| Source commit | `9e814c88adadde88d34142d52b63d220113cb707` |
| Container registry | GitHub Container Registry (GHCR) |
| Published image | `ghcr.io/zohargolano/sample-nodejs:sha-9e814c88adad` |
| Kubernetes namespace | `sample-app` |
| Argo CD Application | `sample-nodejs` |

### Release Workflow

The application version was updated from `1.0.0` to `1.0.1`.

The following files were updated:

- `package.json`
- `package-lock.json`
- `helm/sample-nodejs/Chart.yaml`

The version update was reviewed and merged into the `main` branch.

GitHub Actions then executed application tests, dependency auditing, Semgrep SAST, Docker build, and Trivy scanning.

All three CI jobs completed successfully.

The approved Docker image was published to GHCR with a commit-based image tag.

### Image Promotion

The image tag in `helm/sample-nodejs/values.yaml` was updated from:

```text
sha-554122410ff7
```

to:

```text
sha-9e814c88adad
```

The promotion was performed through GitHub and Argo CD, without manually changing the Kubernetes Deployment.

### Deployment Verification

The following commands were executed:

```bash
kubectl get application sample-nodejs -n argocd

kubectl rollout status deployment/sample-nodejs -n sample-app

kubectl get pods -n sample-app \
  -o custom-columns='NAME:.metadata.name,IMAGE:.spec.containers[0].image'
```

Observed results:

**Argo CD:**

```text
NAME            SYNC STATUS   HEALTH STATUS
sample-nodejs   Synced        Healthy
```

**Kubernetes Deployment:**

```text
deployment "sample-nodejs" successfully rolled out
```

**Running Pods:**

```text
NAME                            IMAGE
sample-nodejs-5cd64c9df-v7bgc   ghcr.io/zohargolano/sample-nodejs:sha-9e814c88adad
sample-nodejs-5cd64c9df-vzg2g   ghcr.io/zohargolano/sample-nodejs:sha-9e814c88adad
```

### Result

**PASSED**

The new container image was successfully deployed to both application Pods.

Argo CD reported the Application as `Synced` and `Healthy`, and Kubernetes confirmed successful rollout completion.

The test demonstrates a controlled release process that separates image building and security validation from deployment promotion through GitOps.
