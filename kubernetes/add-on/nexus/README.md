# Deploy and Configure Nexus Repository with Helm

Deploy Sonatype Nexus Repository with persistent storage and HTTPS ingress. Keep this file and `values.yaml` in the same directory and run the commands from there.

**Versions:** Nexus `3.96.1`, chart `64.2.0`.

> This guide uses the deprecated Sonatype chart from the recorded deployment. Use an empty volume for a fresh installation. Existing OrientDB data must be migrated first; compare existing release values before applying this example to a running deployment.

**Prerequisites:** Kubernetes access, Helm 3, a working StorageClass, an ingress controller, and cert-manager with a configured ClusterIssuer. Point your hostname to the ingress controller.

## Step 1: Check and Update the Values File

```bash
kubectl config current-context
kubectl get storageclass
vim values.yaml
```

Update the following for your environment:

- Replace `nexus.example.com` in both the ingress host and TLS hosts.
- Set the correct StorageClass, ingress class, and ClusterIssuer.
- Adjust the volume size and CPU/memory settings if needed.

The example uses `cinder-csi`, `nginx`, and `letsencrypt-prod`. These dependencies must already exist. Without cert-manager, remove its annotation and provide the configured TLS Secret yourself.

## Step 2: Add the Helm Repository

```bash
helm repo add sonatype https://sonatype.github.io/helm3-charts/
helm repo update sonatype
helm search repo sonatype/nexus-repository-manager --version 64.2.0
```

## Step 3: Deploy Nexus with Helm

```bash
helm upgrade --install nexus sonatype/nexus-repository-manager \
  --version 64.2.0 \
  --namespace nexus \
  --create-namespace \
  --values values.yaml \
  --wait \
  --timeout 25m
```

This installs a new release or updates an existing one. A chart deprecation warning is expected. `Recreate` causes downtime during restarts and upgrades.

## Step 4: Check the Deployment

```bash
kubectl get all -n nexus
kubectl get pvc,ingress -n nexus
kubectl rollout status deployment/nexus-nexus-repository-manager -n nexus
kubectl logs deployment/nexus-nexus-repository-manager -n nexus --tail=100
```

When using cert-manager, check the certificate:

```bash
kubectl get certificate nexus-tls -n nexus
```

Wait for a Ready pod and `READY=True` on the certificate. Open `https://nexus.example.com`, replacing the hostname with your own.

## Step 5: Sign In and Complete Setup

For a fresh installation, retrieve the initial administrator password:

```bash
kubectl exec deployment/nexus-nexus-repository-manager -n nexus -- \
  cat /nexus-data/admin.password
```

Sign in as `admin`, change the password, and complete setup. Review anonymous access and Community Edition usage limits. An existing deployment keeps its existing password; the initial-password file may no longer exist. Never commit passwords or database exports to Git.

## Step 6: Enable Docker Repositories (Optional)

In `values.yaml`, set `nexus.docker.enabled: true`, uncomment `registries`, and replace their hostnames. Configure DNS and repeat Step 3.

In the Nexus UI, create `docker (hosted)` repositories with HTTP connectors matching ports `8082` and `8083`. Configure the Docker Bearer Token Realm and repository permissions. Helm creates the Kubernetes routing resources, but it does not create repositories inside Nexus. HTTPS terminates at the ingress controller.

Test with your actual registry hostname:

```bash
docker login registry.example.com
```

Then test pushing and pulling a permitted image.

## Update the Deployment

Edit `values.yaml` and repeat Step 3. Before changing the application version, check the [official upgrade path](https://help.sonatype.com/en/nexus-repository-upgrade-paths.html) and take a consistent backup. A Helm rollback does not reverse database changes; PVC retention is not a backup.

```bash
helm history nexus -n nexus
kubectl get deployment nexus-nexus-repository-manager -n nexus \
  -o jsonpath='{.spec.template.spec.containers[*].image}{"\n"}'
```

The chart's `APP VERSION` metadata can show an older version; the image and Nexus UI identify the running application version.
