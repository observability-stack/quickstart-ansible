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

* Ubuntu 22.04 as the host operating system
* Ansible >= 2.9
* Ansible `kubernetes.core` collection

## Usage

1. Modify the `variables.yaml` file to match your environment specifications and preferences:

    * ``observers`` and ``observees``: Define the properties of your kind clusters such as clusterName, podSubnet, serviceSubnet, and dnsDomain.    

2. Install `kubernetes.core` collection from ansible-galaxy

```
ansible-galaxy collection install kubernetes.core 
```
3. Run the installer

```
ansible-playbook -i hosts -u <username> installer.yaml 
```
4. Validate your installation. ``KUBECONFIG`` files for each cluster will be under `/root/.kube/` directory

```
root@observability-stack:~# kind get clusters
observee01
observee02
observer
```

```
root@observability-stack:~# export KUBECONFIG=/root/.kube/observer-config

root@observability-stack:~# kubectl get clusters -n fleet-default
NAME         BUNDLES-READY   NODES-READY   SAMPLE-NODE                LAST-SEEN              STATUS
observee01   1/1             1/1           observee01-control-plane   2024-04-25T16:45:09Z
observee02   1/1             1/1           observee02-control-plane   2024-04-25T16:45:03Z
observer     1/1             1/1           observer-control-plane     2024-04-25T16:58:54Z
```

```
root@observability-stack:~# kubectl get ClusterGroup -n fleet-default
NAME                CLUSTERS-READY   BUNDLES-READY   STATUS
observee-clusters   2/2              2/2
observer-clusters   1/1              1/1
```

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
## License
Licensed under the GNU GENERAL PUBLIC LICENSE Version 3.
