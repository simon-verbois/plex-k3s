# Deploying Plex Media Server on K3s/Kubernetes

This repository contains a set of Kubernetes manifests for deploying Plex Media Server on a K3s cluster (or any other Kubernetes cluster).

The configuration is designed to be simple and utilizes advanced features such as hardware transcoding via NVIDIA GPUs.

## Features

- Simple deployment via `kubectl apply`.
- Data persistence managed by `PersistentVolumeClaims`.
- Centralized configuration in a `ConfigMap`.
- Isolated secrets management (not committed to the repository).
- Secure exposure via an `Ingress` (tested with Traefik).
- Support for hardware transcoding via NVIDIA GPUs (optional).

## Prerequisites

1. A working Kubernetes cluster (e.g., K3s, k0s, RKE2).
2. The `kubectl` command-line tool configured to access your cluster.
3. An **Ingress Controller** installed in the cluster (e.g., Traefik, NGINX Ingress).
4. A **StorageClass** configured to dynamically provision storage volumes. K3s includes `local-path-provisioner`, which works for this configuration.
5. **(Optional)** For hardware transcoding: an NVIDIA GPU on one of the nodes and the [NVIDIA Device Plugin for Kubernetes](https://github.com/NVIDIA/k8s-device-plugin) installed.

## Installation

### 1. Clone the repository

```bash
git clone https://github.com/simon-verbois/plex-k3s-configuration
cd plex-k3s-configuration
```

### 2. Copy the files

Make your own copy of the templates files with this command.

```bash
for file in *.yaml.template; do mv "$file" "${file%.template}"; done
```

### 3. Customize the configuration

You **must** adapt some files to your own environment before applying them.

- **`02-secrets.yaml`**:
  1. **Get your token:** Go to [https://www.plex.tv/claim](https://www.plex.tv/claim) to generate a new token. It is valid for a few minutes.
  2. **Edit `02-secrets.yaml`** and replace `REPLACE-ME-WITH-YOUR-CLAIM-TOKEN` with the token you just generated.

<br>

- **`03-configmap.yaml`**:
  - Change the time zone to `TZ` if needed.
  - Adjust `PLEX_UID` and `PLEX_GID` to match your user GID.

<br>

- **`04-deployment.yaml`**:
  - Adjust the <b>fsGroup</b> to match your user GID.
  - Edit the `volumes` section at the end of the file. Replace the `hostPath` paths (`/data/disk1`, `/data/md0/media`, etc.) with the actual paths where your media resides on your Kubernetes nodes. 

```yaml
volumes:
# ... other volumes ...
- name: media
hostPath:
path: /path/to/your/movies # <-- EDIT THIS
type: Directory
# Add as many hostPath volumes as needed for your libraries
```

<br>

- **`05-service.yaml`**:
  - Has we use the host mode for the network (for network discovery)), we have to create a dedicated endpoint with your node IP
 
```yaml
endpoints:
  - addresses:
      - "xx.xx.xx.xx" # <-- EDIT THIS
```

<br>

- **`06-ingress.yaml`**:
  - Depending on your ingress, you have to set an annotation for the TLS certificate generation
  - Modify the `host` to use your own domain name.
  - Adapt the ingress annotations to your Ingress Controller if you are not using Traefik or if your `cert-resolver` has a different name. 

```yaml
# ...
  annotations:
    traefik.ingress.kubernetes.io/router.tls.certresolver: your-ingress-certresolver-name
# ...
spec:
rules:
- host: "plex.your-domain.com" # <-- EDIT THIS
# ...
tls:
- hosts:
- "plex.your-domain.com" # <-- EDIT THIS
```

### 4. Deploy Plex

Apply all YAML manifests in a single command from the project root:

```bash
kubectl apply -f .
```

This will create the namespace, PVCs, secret, configmap, deployment, service, and ingress.

### 5. Access Plex

After a few minutes, while the image is downloaded and the pod starts, you should be able to access your Plex instance via the URL you configured in the ingress (e.g., `https://plex.your-domain.com`).

The initial configuration of Plex (adding libraries) will be done via this web interface.

## Maintenance

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

## License

This project is licensed under the MIT License. See the `LICENSE` file for details.