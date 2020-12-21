# Overview

The exam has five weighted domains that cover different topics. They are broken out as shown below:

| Domain | Weight |
|--------|--------|
| Cluster architecture, installation, and configuration | 25% |
| Workloads and scheduling | 15% |
| Services and networking | 20% |
| Storage | 10% |
| Troubleshooting | 30% |

The exam environment is current running v1.19. For any learning that I do, I should use a v1.19 environment.

You can use the following resources during the exam, so study them now!

* Kubernetes docs: https://kubernetes.io/docs/
* Kubernetes GitHub: https://github.com/kubernetes/
* Kubernetes Blog: https://kubernetes.io/blog/

Don't try to access any other sites during the exam! (That means no checking Twitter for the latest update from your favorite K8s superstar.)

## Curriculum

The current curriculum is hosted on [GitHub](https://github.com/cncf/curriculum). There are PDFs for each exam type. The CKA curriculum is broken down by domain.

### Cluster architecture, installation, and configuration

The following objectives live under this domain:

* Manage RBAC
* Use Kubeadm to install a basic cluster
* Manage a highly-available Kubernetes cluster
* Provision underlying infrastructure to deploy a Kubernetes cluster
* Perform a version upgrade on a Kubernetes cluster using Kubeadm
* Implement etcd backup and restore

### Workloads and scheduling

The following objectives live under this domain:

* Understand deployments and how to perform rolling update and rollbacks
* Use ConfigMaps and Secrets to configure applications
* Know how to scale applications
* Understand primitives used to create robust, self-healing, application deployments
* Understand how resource limits can affect Pod scheduling
* Awareness of manifest management and common templating tools

### Services and networking

* Understand host networking configuration on the cluster nodes
* Understand connectivity between Pods
* Understand ClusterIP, NodePort, LoadBalancer service types and endpoints
* Know how to use Ingress controllers and Ingress resources
* Know how to configure and use CoreDNS
* Choose an appropriate container network interface plugin

### Storage

The following objectives live under this domain:

* Understand storage classes, persistent volumes
* Understand volume mode, access modes, and reclaim policies for volumes
* Understand persistent volume claims primitive
* Know how to configure applications with persistent storage

### Troubleshooting

The following objectives live under this domain:

* Evaluate cluster and node logging
* Understand how to monitor applications
* Manage container stdout & stderr logs
* Troubleshoot application failure
* Troubleshoot cluster component failure
* Troubleshoot networking