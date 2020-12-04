# Cluster architecture, installation, and configuration

The following objectives live under this domain:

* Manage RBAC
* Use Kubeadm to install a basic cluster
* Manage a highly-available Kubernetes cluster
* Provision underlying infrastructure to deploy a Kubernetes cluster
* Perform a version upgrade on a Kubernetes cluster using Kubeadm
* Implement etcd backup and restore

## Manage RBAC
Managing RBAC means understanding what the RBAC components are and how they fit together. Right now I know that there are roles, roleBindings, clusterRoles, clusterRoleBindings, and serviceAccounts. Those are the objects I know about. 

When a user or pod (service account) tries to use the Kubernetes API, they have to go through three steps: Authentication (AuthN), Authorization (AuthZ), and Admission Control.

**Authentication** is configured through an *Authenticator module* running on the API server. This doesn't seem to be in scope with the objective, but it's good to know. You can have multiple authentication modules running and they could use items like certificates, passwords, JWTs, etc. There is no `User` object or stored usernames, which is hella-weird, but OK. You do you K8s.

**Authorization** is where we start getting into the world of RBAC. When submitting a request, you need to include the username, requested action, and object affected by the action. K8s will check policies associated with you and determine if there is a policy allowing the action. There are different authorization modules, including ABAC, RBAC, and Webhook. The focus for the exam is on RBAC.

**Admission control** is handled by modules that can either modify or reject a request. Admission controllers do not act on read requests. Their purpose is mostly to check on data about the object being modified, and then make a decision or alteration to the original request. Requests can go through multiple AC modules, and if any one rejects the request, processing stops.

### RBAC notes

The API group that RBAC uses is `rbac.authorization.k8s.io`. The API server needs to be started with the `--authorization-mode=RBAC` flag and value to enable RBAC.

Woo hoo! I was right. There are four API objects for RBAC:

* Role - sets permission in a namespace
* RoleBinding - grants permissions to a list of *subjects*
* ClusterRole - sets permissions in a non-namespace context
* ClusterRoleBinding - grants permissions to a list of *subjects*

#### Roles

What's in this crazy role thing? Let's start with a manifest:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: tacos
  name: taco-pod
rules:
- apiGroups: [""]
  resources: ["pods"]
  resourceNames: [""]
  verbs: ["get","watch","list"]
```

Let's get some more context around the fields in the manifest, especially `apiGroups`, `resources`, and `verbs`.

* apiGroups: These are REST paths to different [API groupings](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.19/#-strong-api-groups-strong-). An example would be `extension`.
* resources: This can be any defined resource type, including custom resources.
* resourceNames: You can directly reference a named resource, but it cannot apply to `create` since the resource does not yet exist.
* verbs: I think these are related to API verbs, but I need to find a comprehensive list *TODO*

Commands for dealing with roles:
  * `kubectl create role`: example - `kubectl create role pod-reader --verb=get --verb=list --verb=watch --resource=pods`

You can reference sub-resources by giving the path from the main resource, for instance `pods/status`. That could be useful for giving a service permissions to check the status of a pod, but not other aspects. 

## Use Kubeadm to install a basic cluster

## Manage a highly-available Kubernetes cluster

## Provision underlying infrastructure to deploy a Kubernetes cluster

## Perform a version upgrade on a Kubernetes cluster using Kubeadm

## Implement etcd backup and restore