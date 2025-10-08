# Deploying Plex Media Server on K3s/Kubernetes

This repository contains a set of Kubernetes manifests for deploying [Plex Media Server](https://www.plex.tv/) on a MicroK8s cluster (tested on single node cluster) it should work any other Kubernetes cluster with some adjustments.

The configuration is designed to separate application configuration, metadata, and the actual media files for better data management.

<br>

## Prerequisites

1. A working Kubernetes cluster (e.g., K3s, k0s, RKE2).
2. The `kubectl` command-line tool configured to access your cluster.
3. An **Ingress Controller** installed in the cluster (e.g., Traefik, NGINX Ingress).
4. A **StorageClass** configured to dynamically provision storage volumes. K3s includes `local-path-provisioner`, which works for this configuration.
5. **(Optional)** For hardware transcoding: an NVIDIA GPU on one of the nodes and the [NVIDIA Device Plugin for Kubernetes](https://github.com/NVIDIA/k8s-device-plugin) installed.

## Installation

### 1. Clone the repository

```bash
git clone https://github.com/simon-verbois/plex-k8s
cd plex-k8s
```

### 2. Rename the files

```bash
for file in *.yaml.template; do mv "$file" "${file%.template}"; done
```

### 3. Custom the configuration files

You have to adapt the yaml file, follow the hint for each file

### 4. Deploy Plex

Apply all YAML manifests in a single command from the project root:

```bash
kubectl apply -f .
```

This will create the namespace, PVCs, secret, configmap, deployment, service, and ingress.

### 5. Access Plex

After a few minutes, while the image is downloaded and the pod starts, you should be able to access your Plex instance via the URL you configured in the ingress (e.g., `https://plex.your-domain.com`).

The initial configuration of Plex (adding libraries) will be done via this web interface.

<br>

## Maintenance

### Updating the Plex Image

The deployment uses the `plexinc/pms-docker:latest` image. To update to the latest version, you can trigger a rolling update of the deployment, which will force Kubernetes to pull the newest image.

```bash
kubectl rollout restart deployment/plex-deployment -n plex
```

You can monitor the progress of the update with:

```bash
kubectl rollout status deployment/plex-deployment -n plex
```

### Database Repair

Just for your information, if one day you encounter an issue with Plex DB, you can try a repair with this well know script.

1. Download the script from the [official repository](https://github.com/ChuckPa/DBRepair).
2. Identify your Plex pod name:
```bash
kubectl get pods -n plex
```
3. Copy the script into the container:
```bash
kubectl cp DBRepair.sh plex/PLEX_POD_NAME:/config/DBRepair.sh -n plex
```
4. Run the script inside the container:
```bash
kubectl exec -it PLEX_POD_NAME -n plex -- /bin/bash /config/DBRepair.sh
```

<br>

## Uninstallation

To remove all the resources created by these manifests, run the following command:

```bash
kubectl delete -f .
```

**Note:** This will also delete the `plex` namespace. The `PersistentVolumeClaim` will be deleted, but the actual data on your storage volume might remain, depending on your `StorageClass` reclaim policy.

<br>

## License

This project is licensed under the MIT License. See the `LICENSE` file for details.