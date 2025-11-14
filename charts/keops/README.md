# Clarus AI Toolkit Helm Chart

## Description

The **Clarus AI Toolkit** chart deploys a complete platform of services on Kubernetes, including components for processing, storage, monitoring, and workflow orchestration. This chart integrates multiple third-party applications and auxiliary services, enabling a cohesive and managed deployment through Helm.

## Main Components

This chart includes the following main dependencies:

| Component                              | Version | Repository                                                                 | Description                                                   |
| -------------------------------------- | ------- | -------------------------------------------------------------------------- | ------------------------------------------------------------- |
| **PostgreSQL**                         | 18.1.3  | [Bitnami](https://charts.bitnami.com/bitnami)                              | Relational database used by Airflow and MLflow.               |
| **Airflow**                            | 1.15.0  | [Apache](https://airflow.apache.org)                                       | Workflow orchestrator with KubernetesExecutor by default.     |
| **MinIO**                              | 17.0.21 | [Bitnami](https://charts.bitnami.com/bitnami)                              | S3-compatible object storage for logs and ML artifacts.       |
| **Redis**                              | 23.2.6  | [Bitnami](https://charts.bitnami.com/bitnami)                              | Used by Airflow (if CeleryExecutor were enabled).             |
| **MLflow**                             | 5.1.17  | [Bitnami](https://charts.bitnami.com/bitnami)                              | Tracks experiments, models, and serves artifacts from MinIO.  |
| **Docker Registry**                    | 3.0.0   | [Twuni](https://twuni.github.io/docker-registry.helm)                      | Private registry for custom Airflow/ML images.                |
| **Kube Prometheus Stack** *(optional)* | 67.7.0  | [Prometheus Community](https://prometheus-community.github.io/helm-charts) | Full monitoring stack (Prometheus + Grafana + Alertmanager).  |
| **DCGM Exporter** *(optional)*         | 4.6.0   | [NVIDIA](https://nvidia.github.io/dcgm-exporter/helm-charts)               | Exports GPU telemetry metrics.                                |
| **NVIDIA Device Plugin** *(optional)*  | 0.17.1  | [NVIDIA](https://nvidia.github.io/k8s-device-plugin)                       | Enables GPU access for pods.                                  |
| **DAG Generator UI (`daggen`)**        | —       | Custom         

## Prerequisites

* Kubernetes >= **1.25**
* Helm >= **3.8**
* Access to the repositories listed above
* Access to the Central DAG Repository, used for Airflow workflow synchronization.
* (Optional) NVIDIA GPU nodes if GPU components are enabled

## Installation

```bash
helm repo add clarus https://CLARUS-Project.github.io/helm_charts
helm dependency update
helm install ai-toolkit clarus/ai-toolkit -n ai-toolkit --create-namespace
```

### Deployment on K3s

This chart has been successfully tested on a K3s cluster, a lightweight Kubernetes distribution ideal for local, edge, and small-scale production environments.
While the deployment was validated on K3s, the chart is fully compatible with any standard Kubernetes distribution, including GKE, EKS, AKS, and self-managed clusters, as it relies exclusively on standard Kubernetes and Helm APIs.

#### Creating a K3s Cluster

The simplest way to create a K3s cluster is to run the installation script on your main node:

```bash
curl -sfL https://get.k3s.io | sh -
```

Once installed, you can verify the cluster status:

```bash
sudo k3s kubectl get nodes
```

For a multi-node cluster, simply join additional worker nodes with the following command (replace `<MASTER_IP>` and `<TOKEN>`):

```bash
curl -sfL https://get.k3s.io | K3S_URL=https://<MASTER_IP>:6443 K3S_TOKEN=<TOKEN> sh -
```

You can retrieve the join token from the master node with:

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

#### Official Documentation

For more details and advanced configurations, refer to the official K3s documentation:
🔗 [https://docs.k3s.io/](https://docs.k3s.io/)


## Customization and Values

The Helm chart provides extensive configuration options through the `values.yaml` file.

### Obtaining Default Values

You can inspect the default configuration directly from the chart repository:

```bash
helm show values clarus/ai-toolkit > values.yaml
```

This command outputs all default parameters into a `values.yaml` file that you can modify to fit your deployment.

### Main Sections to Customize

Key configuration sections in the AI toolkit chart include:

| Section                 | Description                                                         |
| ----------------------- | ------------------------------------------------------------------- |
| `global`                | Global configuration variables shared across subcharts.             |
| `postgresql`            | Database credentials, storage settings, and initialization scripts. |
| `minio`                 | Object storage configuration, including access and secret keys.     |
| `airflow`               | Scheduler, webserver, and executor settings for Airflow.            |
| `redis`                 | Persistence and replica settings for Redis.                         |
| `mlflow`                | MLflow tracking and artifact storage settings.                      |
| `docker-registry`       | Registry credentials and storage configuration.                     |
| `kube-prometheus-stack` | Enable or disable Prometheus and Grafana monitoring.                |
| `dcgm-exporter`         | Enable GPU metrics exporter for NVIDIA hardware.                    |
| `nvidia-device-plugin`  | GPU device exposure configuration.                                  |
| `daggen`                | DAG generation configuration for Airflow pipelines.                 |

### Enabling Optional Components

To enable optional components like the Kube Prometheus Stack or NVIDIA GPU support, set the corresponding `enabled` flags to `true` in your `values.yaml` file:

```yaml
kube-prometheus-stack:
  enabled: true

nvidia-device-plugin:
  enabled: true

dcgm-exporter:
  enabled: true
```

### Airflow DAG Synchronization Configuration

The Airflow component in AI toolkit is designed to automatically load Directed Acyclic Graphs (DAGs) from a central Git repository. This feature is essential, as it enables synchronization between environments and ensures consistency in deployed workflows.

Check the (central DAG repository)[https://github.com/CLARUS-Project/central-repo] for the full documentation on how to structure and manage your DAGs.

The relevant section of the `values.yaml` file is as follows:

```yaml
airflow:
  dags:
    gitSync:
      enabled: true
      repo: "git@gitlab.example.com:your-org/your-repo.git"
      branch: "main"
      subPath: ""
      sshKeySecret: "airflow-ssh-secret"
      wait: 15
```

| Parameter      | Description                                                                                                                                                                                                                         | Example                              |
| -------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------ |
| `enabled`      | Enables Git-Sync for DAGs.                                                                                                                                                                                                          | `true`                               |
| `repo`         | The Git repository URL where your DAGs are stored.                                                                                                                                                                                  | `"git@github.com:myorg/my-dags.git"` |
| `branch`       | The branch to synchronize from.                                                                                                                                                                                                     | `"main"`                             |
| `subPath`      | (Optional) Subdirectory inside the repository containing the DAGs. Leave empty if DAGs are in the root directory.                                                                                                                  | `"airflow/dags"`                     |
| `sshKeySecret` | (Optional) The Kubernetes secret containing the SSH private key used for authentication to private repositories. **This field is only required for private repositories**. If your repository is public, you can simply comment out this line. | `"airflow-ssh-secret"`               |
| `wait`         | Time in seconds Git-Sync waits between sync attempts.                                                                                                                                                                               | `15`                                 |

#### Notes

* If your repository is private, ensure you have created a Kubernetes secret containing your SSH key:

```bash
kubectl create secret generic airflow-ssh-secret --from-file=gitSshKey=/path/to/id_rsa -n <deploy-namespace>
```

* If your repository is public, you can safely comment out the sshKeySecret line:

```yaml
# sshKeySecret: "airflow-ssh-secret"
```

### Applying Custom Values

You can apply your modified configuration using the `-f` flag to specify a local `values.yaml` file:

```bash
helm install ai-toolkit clarus/ai-toolkit -n ai-toolkit --create-namespace -f values.yaml
```

Or by setting individual parameters inline with `--set`:

```bash
helm install ai-toolkit clarus/ai-toolkit -n ai-toolkit --create-namespace \
  --set airflow.dags.gitSync.repo=https://github.com/CLARUS-Project/central-repo.git \
  --set kube-prometheus-stack.enabled=true
```

## Uninstallation

To completely remove the deployment:

```bash
helm uninstall ai-toolkit -n ai-toolkit
```

## Versioning

* **Chart version:** 0.1.0
* **App version:** 1.16.0

## License

This project is distributed under the MIT License, except for third-party components which retain their own licenses.
