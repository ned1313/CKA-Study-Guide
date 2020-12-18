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

There are also three types of subjects. 

* Users - are defined using a string and must map to an object in an authentication module
* Groups - are defined using a string and must map to an object in an authentication module
* ServiceAccounts - are defined within Kubernetes and prefixed with `system:serviceaccount:`

The only restriction on naming groups and users is that the prefix `system` is reserved for Kubernetes subjects. So don't try it! Things could get real wonky. ServiceAccounts can belong to groups, which uses the prefix `system:serviceaccounts:` (note the plural).

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

* **apiGroups**: These are REST paths to different [API groupings](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.19/#-strong-api-groups-strong-). An example would be `extension`.
* **resources**: This can be any defined resource type, including custom resources.
* **resourceNames**: You can directly reference a named resource, but it cannot apply to `create` since the resource does not yet exist.
* **verbs**: I think these are related to API verbs, but I need to find a comprehensive list *TODO*

Commands for dealing with roles:
  * `kubectl create role`: example - `kubectl create role taco-pods --verb=get --verb=list --verb=watch --resource=pods --namespace=tacos`

You can reference sub-resources by giving the path from the main resource, for instance `pods/status`. That could be useful for giving a service permissions to check the status of a pod, but not other aspects. 

#### Role Binding

The next item in RBAC has to be the RoleBinding, which connects the role to an identity. Once again, let's take a look at a sample manifest to get an idea of what's going on inside the RoleBinding.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: taco-pod-binding-ned
  namespace: tacos
subjects:
# You can specify more than one "subject"
- kind: User
  name: ned # "name" is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  # "roleRef" specifies the binding to a Role / ClusterRole
  kind: Role #this must be Role or ClusterRole
  name: taco-pod # this must match the name of the Role or ClusterRole you wish to bind to
  apiGroup: rbac.authorization.k8s.io
```

Alright, let's break things down a bit.

* **kind** is obvious and the metadata specifies where this resource should be created. RoleBindings are namespaced, so we have to give it a name and a namespace. 
* **subjects** looks like you can specify one or more subjects of different kinds. What kinds are supported? Users, groups, and service accounts.
* **roleRef** refers to the role you want to bind. Turns out this can be either a Role or a ClusterRole. If you use a ClusterRole, it will only apply to the namespace defined in the metadata.

Commands for working with RoleBindings:
  * `kubectl create rolebinding`: example - `kubectl create rolebinding taco-pod-binding-ned --role=taco-pod --user=ned --namespace tacos`

#### ClusterRole

A ClusterRole is like the role we looked at before, but it is not restricted to a namespace. You can use a ClusterRole to define permissions on resources that are not namespaced, or provide blanket permissions across an entire cluster and all namespaces within. The classic example is the `cluster-admin`, which can do basically anything when bound to a user.

What does a manifest for a ClusterRole look like? (Hint: very similar to a Role)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  # "namespace" omitted since ClusterRoles are not namespaced
  name: secret-reader
rules:
- apiGroups: [""]
  # at the HTTP level, the name of the resource for accessing Secret
  # objects is "secrets"
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]
```

The role above would be able to monitor and access Kubernetes secrets across all namespaces (secrets are a namespaced resource). We could also add a cluster resource like a node, which is not namespaced. The individual fields are virtual identical to the Role, with the exception being the lack of a namespace in the metadata area.

You would create a ClusterRole imperatively by using:
  * `kubectl create clusterrole`: example - `kubectl create clusterrole secret-reader --resource=secrets --verb=get,watch,list`

##### ClusterRole Aggregation

ClusterRoles can also aggregate the settings of one or more ClusterRoles into a single role. You can define permissions on resources separately and then mix and match a ClusterRole you need. A primary use case would be granting permissions on Custom Resource Definitions. When a new CRD is being defined, you could also define a ClusterRole and add it to existing ClusterRoles, without altering those ClusterRoles.

Kubernetes uses aggregation for some of the user-facing roles that are created by default. The aggregation setting uses labels to decide which ClusterRoles to include in the aggregated ClusterRole. Let's take a look at the `admin` ClusterRole that is created by default.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: admin
aggregationRule:
  clusterRoleSelectors:
  - matchLabels:
      rbac.authorization.k8s.io/aggregate-to-admin: "true"
rules: []
```

Any ClusterRole created that has the label `rbac.authorization.k8s.io/aggregate-to-admin` set to `"true"` will be included in this aggregate ClusterRole. So if I add a new CRD called `burrito` and want to allow someone with the `admin` ClusterRole bound to administer `burrito` customer resources, I can create the following ClusterRole:

```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: aggregate-burrito-admin
  labels:
    # Add these permissions to the "view" default role.
    rbac.authorization.k8s.io/aggregate-to-admin: "true"
rules:
- apiGroups: [""]
  resources: ["burritos"]
  verbs: ["create", "delete", "patch","update","get","list","watch"]
```

You could even create a ClusterRole with view permissions and another with modify permissions and label both for inclusion in the admin ClusterRole. This is a pretty cool tool! There is nothing like this for regular rules.

#### ClusterRoleBinding

Just like we saw with the RoleBinding, you can bind an identity (user, group, or service account) to a ClusterRole. Once again, since there is no namespacing for ClusterRoles, you don't specify one in the configuration.

Here's an example manifest:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-secrets-global-ned
subjects:
- kind: User
  name: ned # Name is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

The spec is almost the same as the RoleBinding, so no need to get into much detail here. You could accomplish the same goal with the following command:
  * `kubectl create clusterrolebinding`: example - `kubectl create clusterrolebinding read-secrets-global-ned --clusterrole=secret-reader --user=ned`

Excellent. What else do I need to know about RBAC? Nothing, right? WRONG!

#### Default roles

Kubernetes creates a set of default roles and role-bindings when you set it up. The definition of the default roles are refreshes when Kubernetes starts up, and will overwrite any changes or modifications you have made. Sooooo, maybe don't edit those roles. The process is called auto-reconciliation, and it not only helps to keep settings current, it also updates ClusterRoles and ClusterRoleBindings when you upgrade or patch your cluster.

If you look at one of the default roles, you're going to see this annotation: `rbac.authorization.kubernetes.io/autoupdate: "true"`. You can alter the value to `"false"` and Kubernetes will stop auto-reconciling the role. **DANGER** - yeah, you probably shouldn't do this unless you're 100% sure, because you can break your cluster next time it boots.

There are five categories of roles to remember:

* API discovery - Default role bindings authorize unauthenticated and authenticated users to read API information that is deemed safe to be publicly accessible. These roles will have the `system` prefix associated with them, like `system:discovery`.
* User-facing - These roles do not have the `system` prefix and are intended to be associated with users and bound at the cluster or namespace level. Many of these ClusterRoles use aggregation, such as the `admin` role we looked at before.
* Core component - These roles do begin with the `system` prefix and they are for the core components of Kubernetes: kube-scheduler, volume-scheduler, kube-controller-manager, node, and node-proxier.
* Other component - These roles begin with the `system` prefix and honestly, the "Other" labeling seems a bit arbitrary. I'm sure there's a good reason they aren't part of the core, but I can't discern one.
* Built-in controllers - The controller manager will bind these roles to the service account of controllers running on Kubernetes, or to itself if those controllers are not using a service account. All of these roles are prefixed with `system:controller:`.

## Use Kubeadm to install a basic cluster

## Manage a highly-available Kubernetes cluster

## Provision underlying infrastructure to deploy a Kubernetes cluster

## Perform a version upgrade on a Kubernetes cluster using Kubeadm

## Implement etcd backup and restore