# Kubernetes Week 1 Project

## Rancher, Traefik, and GitLab Runner on Kubernetes

This repository contains the manifests, Helm values, installation commands, reports, and verification screenshots for my first-week Kubernetes bootcamp assignment.

The goal of the project was to deploy the following components on a Kubernetes cluster:

- Traefik as the Ingress Controller
- Rancher as the Kubernetes management platform
- GitLab Runner with the Kubernetes executor

The final deployment provides HTTPS access to Rancher and shows the GitLab Runner as `Online` in GitLab.

## Cluster topology

| Host | Role | IP address |
|---|---|---|
| `server04` | Control Plane | `192.168.122.204` |
| `server02` | Worker Node | `192.168.122.174` |
| `server03` | Worker Node | `192.168.122.230` |

I ran the administrative `kubectl` and `helm` commands on `server04`. Kubernetes scheduled the application Pods on the worker nodes.

## Software versions

| Component | Version |
|---|---|
| Kubernetes | `v1.36.4` |
| Container runtime | `containerd 2.2.1` |
| Traefik | `v3.7.11` |
| Rancher Helm chart | `2.15.0` |
| Rancher application | `v2.15.0` |
| GitLab Runner Helm chart | `0.92.0` |
| GitLab Runner | `19.3.0` |

## Repository structure

```text
kubernetes-week1/
├── manifests/
│   ├── traefik/
│   │   ├── namespace.yaml
│   │   ├── rbac.yaml
│   │   ├── ingressclass.yaml
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   ├── rancher/
│   │   ├── rancher-values.yml
│   │   └── rancher-2.15.0.tgz
│   └── gitlab-runner/
│       ├── runner-values.yml
│       ├── gitlab-runner-rendered.yml
│       └── gitlab-runner-0.92.0.tgz
├── reports/
│   └── kubernetes-week1-manifests-report-fa.md
├── screenshots/
│   ├── 01-nodes-and-pods.png
│   ├── 02-services-and-ingress.png
│   ├── 03-helm-list.png
│   ├── 04-rancher-ui.png
│   └── 05-gitlab-runner-online.png
├── commands.txt
└── README.md
```

The exact filenames in `manifests/traefik/` can differ if the Traefik resources are stored in one combined YAML file.

## Architecture

### Traefik

Traefik runs in the `traefik` namespace and acts as the cluster's Ingress Controller. Its Kubernetes resources include:

- Namespace
- ServiceAccount
- ClusterRole and ClusterRoleBinding
- IngressClass
- Deployment
- NodePort Service

The Traefik Service exposes the following ports:

| Protocol | Service port | Container port | NodePort |
|---|---:|---:|---:|
| HTTP | `80` | `8000` | `30080` |
| HTTPS | `443` | `8443` | `30443` |

Traefik watches Kubernetes Ingress resources whose class is `traefik`. It receives the Rancher HTTPS request and forwards it to the Rancher Service.

### Rancher

Rancher runs in the `cattle-system` namespace. I installed it using the local Rancher Helm chart and `rancher-values.yml`.

The configured hostname is:

```text
192.168.122.204.sslip.io
```

I used one Rancher replica because the lab worker nodes have limited disk and memory resources.

TLS is provided using Kubernetes Secrets:

- `tls-rancher-ingress` contains the TLS certificate and private key.
- `tls-ca` contains the CA certificate.

The Rancher Ingress uses the `traefik` IngressClass and the secure Traefik entry point.

### GitLab Runner

GitLab Runner runs in the `gitlab-runner` namespace. I installed it using Helm and configured the Kubernetes executor.

The Runner configuration uses:

- One Runner Manager replica
- One concurrent job
- Kubernetes executor
- Non-privileged job containers
- CPU and memory requests and limits
- A separate Kubernetes Secret for the authentication token

The required result for this assignment is that the Runner Manager Pod is running and the Runner is shown as `Online` in GitLab. A CI/CD pipeline is not required by the assignment.

## Prerequisites

Before deployment, the following tools must be available on `server04`:

```bash
kubectl version --client
helm version
```

The cluster must also have three Ready nodes:

```bash
kubectl get nodes -o wide
```

## Deployment

All commands in this section are executed on `server04`.

### 1. Deploy Traefik

Apply the Traefik manifests:

```bash
kubectl apply -f manifests/traefik/
```

Wait for Traefik:

```bash
kubectl rollout status deployment/traefik \
  -n traefik \
  --timeout=5m
```

Verify the resources:

```bash
kubectl get pods,services -n traefik -o wide
kubectl get ingressclass
```

### 2. Prepare Rancher TLS Secrets

The certificate and private key must not be committed to the repository. Create the Secrets from local certificate files:

```bash
kubectl create namespace cattle-system \
  --dry-run=client -o yaml |
kubectl apply -f -
```

```bash
kubectl create secret tls tls-rancher-ingress \
  -n cattle-system \
  --cert=/secure/path/tls.crt \
  --key=/secure/path/tls.key \
  --dry-run=client -o yaml |
kubectl apply -f -
```

If a private CA is used, create the CA Secret without adding the CA file to the repository:

```bash
kubectl create secret generic tls-ca \
  -n cattle-system \
  --from-file=cacerts.pem=/secure/path/ca.crt \
  --dry-run=client -o yaml |
kubectl apply -f -
```

### 3. Install Rancher

Store the Bootstrap password temporarily in a shell variable:

```bash
read -s -p "Temporary Rancher admin password: " \
  RANCHER_BOOTSTRAP_PASSWORD
echo
```

Install Rancher using the local chart:

```bash
cd manifests/rancher

helm upgrade --install rancher \
  ./rancher-2.15.0.tgz \
  --namespace cattle-system \
  --values rancher-values.yml \
  --set-string bootstrapPassword="$RANCHER_BOOTSTRAP_PASSWORD" \
  --wait \
  --timeout 20m
```

Remove the password from the shell after installation:

```bash
unset RANCHER_BOOTSTRAP_PASSWORD
```

Verify Rancher:

```bash
helm list -n cattle-system

kubectl rollout status deployment/rancher \
  -n cattle-system \
  --timeout=5m

kubectl get pods,services,ingress \
  -n cattle-system \
  -o wide
```

Test Rancher HTTPS access:

```bash
curl -kI --max-time 20 \
  https://192.168.122.204.sslip.io
```

The expected result is an HTTP success response such as:

```text
HTTP/2 200
```

### 4. Create the GitLab Runner Secret

The GitLab Runner authentication token must be created in the GitLab project and must not be stored in `runner-values.yml`.

Read the token without displaying it:

```bash
read -s -p "GitLab Runner authentication token: " \
  GITLAB_RUNNER_TOKEN
echo
```

Create the Namespace and Secret:

```bash
kubectl create namespace gitlab-runner \
  --dry-run=client -o yaml |
kubectl apply -f -
```

```bash
kubectl create secret generic gitlab-runner-secret \
  -n gitlab-runner \
  --from-literal=runner-token="$GITLAB_RUNNER_TOKEN" \
  --from-literal=runner-registration-token="" \
  --dry-run=client -o yaml |
kubectl apply -f -
```

Remove the token from the shell:

```bash
unset GITLAB_RUNNER_TOKEN
```

### 5. Install GitLab Runner

```bash
cd ../gitlab-runner

helm upgrade --install gitlab-runner \
  ./gitlab-runner-0.92.0.tgz \
  --namespace gitlab-runner \
  --values runner-values.yml \
  --wait \
  --timeout 15m
```

Verify the Runner deployment:

```bash
helm list -n gitlab-runner

kubectl get deployment,pods \
  -n gitlab-runner \
  -o wide
```

The Runner must also appear as `Online` in the GitLab project under:

```text
Settings > CI/CD > Runners
```

## Final verification

I used the following commands to verify the complete deployment:

```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get services -A -o wide
kubectl get ingress -A
helm list -A
```

I also checked for node disk pressure:

```bash
kubectl get nodes \
  -o custom-columns='NAME:.metadata.name,READY:.status.conditions[?(@.type=="Ready")].status,DISK-PRESSURE:.status.conditions[?(@.type=="DiskPressure")].status'
```

The expected final state is:

- All nodes show `Ready=True`.
- Active Traefik, Rancher, and GitLab Runner Pods are `Running` and Ready.
- Rancher returns an HTTPS success response.
- GitLab Runner is displayed as `Online` in GitLab.

## Screenshots

The `screenshots/` directory should contain evidence of:

1. Kubernetes nodes and Pods
2. Services and Ingress resources
3. Installed Helm releases
4. Rancher web interface
5. GitLab Runner displayed as Online

The failed test pipeline is not part of the assignment and does not need to be included.

## Security notes

The following information is intentionally excluded from the repository and ZIP file:

- GitLab Runner authentication token
- Rancher Bootstrap password
- TLS private keys
- kubeconfig files
- Shell history
- Exported Kubernetes Secret contents

Before creating the final archive, I scan the project for exposed Runner tokens and private keys.

## Troubleshooting performed

The lab network experienced TLS handshake timeouts when pulling container images. Where necessary, I downloaded the official images on the Parrot workstation, transferred the image archives temporarily to the worker nodes, imported them into the `k8s.io` containerd namespace, and then removed the temporary archives.

The worker nodes also experienced ephemeral-storage pressure. I extended their LVM root logical volumes and verified the `DiskPressure` node condition before completing the deployment.

Temporary image archives and private credentials are not included in this repository.

## Result

The project was completed with the following results:

- Traefik Ingress Controller deployed successfully
- Rancher `v2.15.0` deployed successfully
- Rancher accessible over HTTPS at `192.168.122.204.sslip.io`
- GitLab Runner `19.3.0` deployed successfully
- GitLab Runner shown as Online in GitLab

