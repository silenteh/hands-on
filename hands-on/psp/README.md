### Pod Security Policy (WORK IN PROGRESS)
Defines a set of specifications (policies) a Pod needs to meet in order to be accepted in the cluster.  
**IMPORTANT** Pod security policy is a `Cluster` resource, not a namespace one.  
The policies are verified by a dedicated **optional** admission controller. Enabling the controller, without a Pod Security Policy defined will **prevent** any Pods from being scheduled.  

>**WARNING**: since `psp` is a cluster wide resource, if you are sharing a cluster, you might have name collisions and/or unexpected results from this hands on.

#### PSP Ordering
A cluster can have multiple Pod Security Policies, so how does Kubernetes evaluate them ? In which order ?
1. Kubernetes gives highest priority to the policies which do no mutate the Pods. If there are multiple policies which behave in this way, then the order does not matter.
2. If the Pod is mutated the alphabetical order of the policy is considered and the first policy which meets the specifications is selected.

---
#### Creating Pod Security Policies
We are going to create a policy where we block the scheduling of privileged pods.  
Create a local file called: `psp.yaml`
```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: unprivileged
spec:
  privileged: false # Prevents privileged pods
  # PSP require setting the following mandatory fields
  seLinux:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  runAsUser:
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  volumes:
  - '*'
```

1. Create the policy in your cluster: `kubectl create -f psp.yaml`
2. Verify the policy was created: `kubectl get psp` 
---

#### Enabling policies

When a Pod Security Policy resource is created, it does _absolutely_ nothing.  
In order to enforce it, you need to authorize the usage of the policy to either:
- The user/service account requesting the scheduling of resources
- The service account the Pod runs with. (Either the `default` service account of the namespace or a dedicated one)

RBAC is used to enable the Pod Security Policies.  
Given the flexibility of RBAC, one can bind the `(Custer)Role` to either the whole cluster or specific namespaces.

1. Create the `(Custer)Role` file named: `psp_role.yaml`
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: unprivileged_psp
rules:
- apiGroups: ['policy']
  resources: ['podsecuritypolicies']
  verbs:     ['use']
  resourceNames:
  - unprivileged
```
2. Verify that the role was created: `kubectl get clusterroles | grep psp`

##### Enable the PSP for a specific namespace only
1. Use a `RoleBinding` with the following name `psp_ns.yaml` and content:  
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: psp-ns
roleRef:
  kind: ClusterRole
  name: unprivileged_psp
  apiGroup: rbac.authorization.k8s.io
subjects:
# Authorize all authenticated users in a namespace. (both users and service accounts)
- kind: Group
  apiGroup: rbac.authorization.k8s.io
  name: system:authenticated
```
2. Apply the RoleBinding to your namespace: `kubectl apply -f psp_ns.yaml -n myspace`

