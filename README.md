# Sample Node.js Application — Targil2026

A small, stateless Express application used to demonstrate container delivery, Kubernetes deployment, security-gated CI/CD, and declarative GitOps in the **Targil2026** lab.

**Application version:** `1.0.1`  
**Helm chart version:** `0.2.1`  
**Helm `appVersion`:** `1.0.1`

- **Application repository:** [ZoharGolano/sample-nodejs](https://github.com/ZoharGolano/sample-nodejs)
- **Infrastructure repository:** [ZoharGolano/targil2026](https://github.com/ZoharGolano/targil2026)
- **Detailed GitOps documentation:** [gitops/README.md](gitops/README.md)

## Application

The service runs on Node.js with Express and exposes the following endpoints:

| Endpoint | Purpose |
| --- | --- |
| `/my-app` | Demonstration endpoint (returns `Hello, World!`) |
| `/ready` | Kubernetes readiness probe |
| `/live` | Kubernetes liveness probe |
| `/metrics` | Prometheus-format application metrics (`prom-client`) |

The application port is configurable through the `PORT` environment variable; the deployed container uses port `8080`.

### Run locally

**Prerequisites:** Node.js 22 and npm.

```bash
npm ci
npm test
PORT=8080 node app.js
```

In another terminal:

```bash
curl -i http://127.0.0.1:8080/my-app
curl -i http://127.0.0.1:8080/ready
curl -i http://127.0.0.1:8080/live
curl -s http://127.0.0.1:8080/metrics | head
```

> The current `package.json` defines `npm test` but does **not** define an `npm start` script. Use `node app.js` to start the service.

## Repository structure

```text
sample-nodejs/
├── app.js                         # Express application and endpoints
├── test/                          # Node.js tests
├── package.json                   # Application dependencies and version
├── package-lock.json              # Locked dependency versions
├── Dockerfile                     # Container image build
├── .dockerignore
├── .github/workflows/ci-cd.yml    # CI/CD and security checks
├── helm/sample-nodejs/            # Application Helm chart
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
├── gitops/
│   ├── application.yaml           # Declarative Argo CD Application
│   └── README.md                  # GitOps setup, testing and release history
└── README.md
```

## Container image

The multi-stage `Dockerfile` builds a Node.js application image with production dependencies and runs the application as a non-root user (`1000:1000`). The runtime container listens on port `8080`.

To build and test an image locally:

```bash
docker build -t sample-nodejs:local .
docker run --rm -p 8080:8080 sample-nodejs:local
```

Then visit `http://127.0.0.1:8080/my-app` or use the curl checks above.

Production release images are published to **GitHub Container Registry (GHCR)** using commit-based tags of the form:

```text
ghcr.io/zohargolano/sample-nodejs:sha-<first-12-characters-of-commit>
```

The image validated and promoted for application version `1.0.1` is:

```text
ghcr.io/zohargolano/sample-nodejs:sha-9e814c88adad
```

## Helm deployment

The application uses a **Deployment**, rather than a StatefulSet, because it is stateless: it does not need stable Pod identities, per-replica persistent volumes, or ordered startup and shutdown.

The chart at [`helm/sample-nodejs/`](helm/sample-nodejs/) defines:

| Setting | Configured behavior |
| --- | --- |
| Replicas | `2` |
| Service | `ClusterIP`, port `80` → container port `8080` |
| Ingress | Traefik, host `sample-nodejs.test` |
| Readiness/liveness | `/ready` and `/live` |
| CPU request / limit | `50m` / `250m` |
| Memory request / limit | `64Mi` / `256Mi` |
| Runtime security | Non-root user, privilege escalation disabled, capabilities dropped |
| Service account | Token automount disabled |

Render and validate the chart **without changing the cluster**:

```bash
helm lint helm/sample-nodejs
helm template sample-nodejs helm/sample-nodejs --namespace sample-app \
  > /tmp/sample-nodejs-rendered.yaml
```

The lab deployment is managed by **Argo CD**. After initial GitOps bootstrap, do not manually upgrade the Helm release to promote an application version: update the image tag in Git and allow Argo CD to reconcile the change.

## CI/CD and security gates

The workflow is defined in [`.github/workflows/ci-cd.yml`](.github/workflows/ci-cd.yml). Pull requests and changes to `main` run automated validation, including:

1. Application tests (`npm test`) and dependency auditing (`npm audit`).
2. **Semgrep** static application security testing (SAST).
3. Docker image build and **Trivy** vulnerability scanning. The Trivy gate fails the relevant CI job on `HIGH` or `CRITICAL` findings.

On successful eligible pushes to `main`, the workflow publishes the tested image to GHCR with a commit-SHA-based tag. Pull request validation does **not** publish an image.

A newly published image is **not automatically deployed merely because it exists in GHCR**. Deployment promotion is a separate, reviewed Git change to the Helm image tag, reconciled by Argo CD.

## GitOps with Argo CD

The Argo CD Application manifest is maintained at [`gitops/application.yaml`](gitops/application.yaml). It references the Helm chart in this repository's `main` branch and deploys into the Kubernetes namespace `sample-app`. Automated synchronization, pruning, and self-healing are enabled.

GitOps delivery flow:

```text
Developer change / pull request
            |
            v
GitHub Actions: tests + audit + Semgrep + build + Trivy
            |
            v
Merge to main -> validated commit-tagged image in GHCR
            |
            v
Reviewed Helm values.yaml image-tag update -> merge to main
            |
            v
Argo CD automatic reconciliation
            |
            v
K3s Deployment rolling update (2 application Pods)
```

The application's host-based Traefik ingress uses `sample-nodejs.test`. For local lab testing from the Ubuntu host, the host header can be provided directly:

```bash
curl -i -H 'Host: sample-nodejs.test' http://192.168.122.11/my-app
```

`192.168.122.11` is the documented K3s lab node IP, not a general-purpose production endpoint.

### Cluster verification

Run these commands from a workstation with a working `kubectl` context for the lab cluster:

```bash
kubectl get application sample-nodejs -n argocd
kubectl rollout status deployment/sample-nodejs -n sample-app
kubectl get deployment sample-nodejs -n sample-app
kubectl get pods -n sample-app \
  -o custom-columns='NAME:.metadata.name,IMAGE:.spec.containers[0].image'
kubectl get ingress,svc -n sample-app
```

**Verified release `1.0.1`:** Argo CD reported `Synced` / `Healthy`, the Deployment rollout succeeded, and both Pods were observed running `ghcr.io/zohargolano/sample-nodejs:sha-9e814c88adad`.

For GitOps bootstrapping, screenshots, troubleshooting, and the detailed release verification, refer to [gitops/README.md](gitops/README.md).

## Versioning and release promotion

Application and chart versions are tracked independently:

- `package.json` and `package-lock.json`: application version (`1.0.1`).
- `helm/sample-nodejs/Chart.yaml` `appVersion`: packaged application version (`1.0.1`).
- `helm/sample-nodejs/Chart.yaml` `version`: chart version (`0.2.1`).
- `helm/sample-nodejs/values.yaml` `image.tag`: explicitly approved image commit tag.

The verified `1.0.1` release process was:

1. Bump the application version and Helm chart metadata in a pull request.
2. Merge after the CI checks succeed; confirm the newly tagged image is present in GHCR.
3. Open a separate pull request updating **only the desired image tag** in Helm values.
4. Merge and let Argo CD synchronize; verify the rolling update and the running Pod images.
5. Record the test evidence in [gitops/README.md](gitops/README.md).

Pinning the tested image tag makes deployments reviewable and avoids unintentionally tracking a mutable `latest` tag.

## Scope and limitations

- The deployment is a **local K3s laboratory**, not a production availability or security claim.
- The three K3s nodes run as VMs on the same physical Ubuntu host; they do not provide physical-host redundancy.
- The MongoDB lab service belongs to the separate infrastructure repository and is **not a required dependency** of this sample Express application.
- Do not commit passwords, access tokens, Kubernetes Secrets, private keys, or administrator kubeconfigs.
- Image vulnerability scans reduce risk but do not guarantee an image has no vulnerabilities.

## Further documentation

- [GitOps installation, validation, troubleshooting and v1.0.1 verification](gitops/README.md)
- [Argo CD Application definition](gitops/application.yaml)
- [Application Helm chart](helm/sample-nodejs/)
- [Infrastructure architecture and K3s/MongoDB documentation](https://github.com/ZoharGolano/targil2026)
