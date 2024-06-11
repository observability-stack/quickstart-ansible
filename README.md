# Observability Stack Quickstart with Ansible and Kind

## Introduction
This project provides an Ansible installer to help users quickly set up Kubernetes clusters using [Kind (Kubernetes in Docker)](https://kind.sigs.k8s.io/) and deployment of a Fleet components (controller and agents) to respective kind clusters, making it ideal for development or testing of [Observability Stack](observability-stack.io).

The installer supports both local and remote virtual machine installation through an inventory/hosts file. By default, the configuration is set for a single-node setup, where both ``observer`` and ``observee`` clusters are hosted on the same (virtual) machine. However, it can be expanded to a multi-node setup with each ``observer`` and ``observee`` clusters on its own virtual machine by making minor modifications.

An example inventory file should be like this;

```
[observability-stack]
192.169.1.10
```

Regardless of the inventory file and the number of hosts, the `observer` and `observee` clusters and their configurations are managed by the `variables.yaml` file. Detailed explanation for each variable is under the [Variables](#Variables) section. 

## Requirements

### Software

* Ubuntu 22.04 as the host operating system.
* Ansible >= 2.9
* Ansible `kubernetes.core` collection.

### Hardware

* **CPU:** Minimum of 4 vCPUs.
* **Memory:** At least 8 GB of RAM; 16 GB is preferable.
* **Storage** Minimum of 50 GB of free disk space, though more is always better.

## Quickstart

1. Clone this repository 

```bash
git clone https://github.com/observability-stack/quickstart-ansible.git
```

2. Run the installer 

**Remotely (with an inventory file):**

Replace `<username>` with your actual username on the remote hosts:

```
ansible-playbook -i hosts -u <username> installer.yaml 
```
**Locally (without an inventory file):**
```
ansible-playbook -i "localhost," installer.yaml
```

3. Validate your installation. ``KUBECONFIG`` files for each cluster will be under `/root/.kube/` directory. 


4. Use `kubectl port-forward` to access Grafana interface

```
kubectl port-forward -n dashboards svc/grafana-service 3000:3000 --address=0.0.0.0
```

## Customizing the Installation

You can follow these steps to customize your installation:

* Modify `variables.yaml`: This file allows you to customize settings for ``` observer``` and ```observee``` cluster configurations such as ```clusterName```, ```podSubnet```, ```serviceSubnet``` etc. You can find detailed descriptions of these variables under the [Variables](#Variables) section.

* Modify Kind Cluster Configuration Templates: Modify the templates directory to customize the Kubernetes clusters set up by kind.You can learn more about these settings in the [Advanced Configuration](#advanced-configuration) section.

Observability Stack Components are deployed using the configurations found in their respective repositories 

* [quickstart-metrics](https://github.com/observability-stack/quickstart-metrics) setup includes:
  * Thanos: Configured with built-in MinIO object storage backend. Check [```values.yaml```](https://github.com/observability-stack/quickstart-metrics/blob/main/tools/thanos/values.yaml) file for details.
  * Grafana: 
    * Grafana Operator: Management and operation of Grafana instance.
    * Grafana Instance: Simple Grafana installation with default settings. Check [```grafana.yaml```](https://github.com/observability-stack/quickstart-metrics/blob/main/tools/grafana/instance/base/grafana.yaml) file for details.
    * Datasources: Datasources (e.g., Thanos) that are configured in the Grafana instance. Check [```grafana-datasources.yaml```](https://github.com/observability-stack/quickstart-metrics/blob/main/tools/grafana/instance/base/grafana-datasource.yaml) file for details.
    * Dashboards: Default dashboards for Observability Stack. Dashboards are stored as json files and imported as ```GrafanaDashboard``` resource. Check [README.md](https://github.com/observability-stack/quickstart-metrics/tree/main/tools/grafana/instance/base/dashboards) file for details and how to add custom dashboards.
  * kube-prometheus: Simple kube-prometheus stack installation with Thanos storage configured. Check [```values.yaml```](https://github.com/observability-stack/quickstart-metrics/blob/main/tools/prometheus/kube-prometheus/values.yaml) file for details.

## Variables

`varibles.yaml` file is a list of observees and observers, and each cluster has different set of variables that has to 

| Parameter          |  Example Value            | Description |
|--------------------|------------------|-------------|
| `metrics.enable` | `true`       | Indicates if metrics are enabled for the Observability Stack. |
| `clusterName`      | `observer`       | Unique name for each cluster in the stack. |
| `podSubnet`        |`10.245.0.0/16`    | The IP range for pod networking. Must allocate different subnet for each cluster |
| `serviceSubnet`    | `10.97.0.0/12`     | The IP range for service networking. Must allocate different subnet for each cluster  |
| `dnsDomain`        | `observer.local`   | Unique internal DNS domain used by the cluster. |
| `fleet`            | `controller` | Indicates that this cluster acts as the Fleet controller. Can be `controller` for observer  or `agent` for observees |
| `observabilityRole`| `observer`     | Role defining the cluster's part in the Observability Stack. Can be `observer` or `observee`. |
| `ports.name`       | `prometheus-grpc`          | Name of the port/service. |
| `ports.port`       | `31901`               | Port number used by the service. |

Example configuration:

```
observability:
  metrics:
    enable: true
observers:
  - clusterName: "observer"
    podSubnet: "10.244.0.0/16"
    serviceSubnet: "10.96.0.0/12"
    dnsDomain: "observer.local"
    fleet: "controller"
    observabilityRole: "observer"
    ports: 
      - name: minio-api
        port: 9000
      - name: prometheus-grpc
        port: 31901
observees:
  - clusterName: "observee01"
    podSubnet: "10.245.0.0/16"
    serviceSubnet: "10.97.0.0/12"
    dnsDomain: "observee01.local"
    fleet: "agent"
    observabilityRole: "observee"
    ports:
      - name: prometheus-grpc
        port: 32901
  - clusterName: "observee02"
    podSubnet: "10.246.0.0/16"
    serviceSubnet: "10.98.0.0/12"
    dnsDomain: "observee02.local"
    fleet: "agent"
    observabilityRole: "observee"
    ports:
      - name: prometheus-grpc
        port: 33901
  # Add more observees as needed
```
## Advanced Configuration

This installer uses [Ansible templating](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_templating.html) to generate dynamic [kind configuration files](https://kind.sigs.k8s.io/docs/user/configuration/#a-note-on-cli-parameters-and-configuration-files) to deploy kind clusters. Since kind itself uses [Kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/) to provision Kubernetes clusters, any configuration option that is either supported by kind or Kubeadm with `kubeadmConfigPatches` can be implemented on the configuration template under the `templates` directory.

Configuration files under `templates` directory can further extended by any configuration option supported by kind such as;

* [Multi-node clusters](https://kind.sigs.k8s.io/docs/user/quick-start/#multi-node-clusters)
* [Ingress types (Contour, Kong, Ingress NGINX)](https://kind.sigs.k8s.io/docs/user/ingress/)
* [LoadBalancer service](https://kind.sigs.k8s.io/docs/user/loadbalancer/) 
* [Kubernetes Version](https://kind.sigs.k8s.io/docs/user/configuration/#kubernetes-version)
* [Extra Mounts](https://kind.sigs.k8s.io/docs/user/configuration/#extra-mounts)
* [Extra Port Mappings](https://kind.sigs.k8s.io/docs/user/configuration/#extra-port-mappings)
* [Kubeadm Config Patches](https://kind.sigs.k8s.io/docs/user/configuration/#kubeadm-config-patches)


An example kind cluster manifest should be like this;

```
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: {{ item.clusterName }}
networking:
  podSubnet: {{ item.podSubnet }}
  serviceSubnet: {{ item.serviceSubnet }}
  apiServerAddress: {{ ansible_default_ipv4.address }}
kubeadmConfigPatches:
- |
  networking:
    dnsDomain: {{ item.dnsDomain }}
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    protocol: TCP
  - containerPort: 443
    protocol: TCP
  - containerPort: 6443
    protocol: TCP
```
## Troubleshooting

### Kind Clusters

```
root@observability-stack:~# kind get clusters
observee01
observee02
observer
```

### Observer Cluster

Source the kubeconfig file and check clusters in fleet-default namespace
```
root@observability-stack:~# export KUBECONFIG=/root/.kube/observer-config
root@observability-stack (observer):~# kubectl get clusters -n fleet-default
NAME         BUNDLES-READY   NODES-READY   SAMPLE-NODE                LAST-SEEN              STATUS
observee01   1/1             1/1           observee01-control-plane   2024-04-25T16:45:09Z
observee02   1/1             1/1           observee02-control-plane   2024-04-25T16:45:03Z
observer     1/1             1/1           observer-control-plane     2024-04-25T16:58:54Z
```
Check cluster groups
```
root@observability-stack (observer):~# kubectl get ClusterGroup -n fleet-default
NAME                CLUSTERS-READY   BUNDLES-READY   STATUS
observee-clusters   2/2              2/2
observer-clusters   1/1              1/1
```
Check GitRepo repositories enrolled to Fleet
```
root@observee01:~# kubectl get gitrepo.fleet.cattle.io -n fleet-default
NAME               REPO                                                            COMMIT                                     BUNDLEDEPLOYMENTS-READY   STATUS
observee-metrics   https://github.com/observability-stack/quickstart-metrics.git   d080424fd1777b9b57d90e6f847631d936621702   2/2
observer-metrics   https://github.com/observability-stack/quickstart-metrics.git   d080424fd1777b9b57d90e6f847631d936621702   4/4
```
Check Pods under metrics namespace
```
root@observability-stack (observer):~# kubectl get pods -n metrics
NAME                                                       READY   STATUS    RESTARTS   AGE
alertmanager-prometheus-monitoring-kube-alertmanager-0     2/2     Running   0          4d4h
prometheus-monitoring-grafana-865668cb4b-mvw9k             3/3     Running   0          4d4h
prometheus-monitoring-kube-operator-f54bfc5f6-kmp8d        1/1     Running   0          4d4h
prometheus-monitoring-kube-state-metrics-bddd5c44d-jbjln   1/1     Running   0          4d4h
prometheus-monitoring-prometheus-node-exporter-g7l5r       1/1     Running   0          4d4h
prometheus-prometheus-monitoring-kube-prometheus-0         3/3     Running   0          4d4h
thanos-compactor-b647c5f99-p7jj5                           1/1     Running   0          4d4h
thanos-minio-74c6cc74df-zq7gj                              1/1     Running   0          4d4h
thanos-query-6ff46d6c9f-p2nqk                              1/1     Running   0          4d4h
thanos-query-frontend-d79b55466-v4s4m                      1/1     Running   0          4d4h
thanos-storegateway-0                                      1/1     Running   0          4d4h
```
### Observee Cluster
```
root@observability-stack:~# export KUBECONFIG=/root/.kube/observee01-config
root@observability-stack (observee01):~# kubectl get pods -n metrics
NAME                                                       READY   STATUS    RESTARTS   AGE
alertmanager-prometheus-monitoring-kube-alertmanager-0     2/2     Running   0          4d4h
prometheus-monitoring-grafana-865668cb4b-mktmd             3/3     Running   0          4d4h
prometheus-monitoring-kube-operator-f54bfc5f6-2hkwn        1/1     Running   0          4d4h
prometheus-monitoring-kube-state-metrics-bddd5c44d-xltsm   1/1     Running   0          4d4h
prometheus-monitoring-prometheus-node-exporter-jz4rx       1/1     Running   0          4d4h
prometheus-prometheus-monitoring-kube-prometheus-0         3/3     Running   0          4d4h
```


## License
Licensed under the GNU GENERAL PUBLIC LICENSE Version 3.
