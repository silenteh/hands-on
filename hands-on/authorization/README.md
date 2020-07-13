### Authorization hands on

#### Create a service account
A service account is authenticated to the Kubernetes API via its signed bearer token.  
Service accounts are usually long term identities created to interact with the Kubernetes API.  
When a service account is created, an associated secret is generated and signed. This is the bearer token and its a JWT signed token.  
Each namespace has a `default` service account and this is the default service account a Pod will be scheduled with.  
You can verify the default service account exists via: `kubectl get serviceaccount`  

We can now create a new service account in your namespace and play around with RBAC.
Here how to create a sevice account (add `-n your_namespace` if needed):  

`kubectl create serviceaccount jenkins`

YOu can check the service account was created:

`kubectl get serviceaccount`

---

#### Create a role for the service account

**WARNING** adjust the namespace accordingly!

Create a local file called: `role-deployment-manager.yaml` and add the following yaml to it.  

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  namespace: default
  name: deployment-manager
rules:
- apiGroups: ["", "extensions", "apps"]
  resources: ["deployments", "replicasets", "pods"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"] # You can also use ["*"]
```

Here the `""` apiGroup is a shortcut for the core (also called _kegacy_) api group: `/api/v1`

Now create the Role: `kubectl create -f role-deployment-manager.yaml`  

You can verify that the Role has been created via: `kubectl get roles deployment-manager -o yaml`

---

#### Bind the role to the service account

Create a local file called `rolebinding-deployment-manager.yaml` and add the following yaml.  

```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: deployment-manager-binding
  namespace: default
subjects:
- kind: ServiceAccount
  name: jenkins
  apiGroup: ""
roleRef:
  kind: Role
  name: deployment-manager
  apiGroup: ""
```

Now create the Role: `kubectl create -f rolebinding-deployment-manager.yaml`  

You can verify that the Role has been created via: `kubectl get rolebindings deployment-manager-binding -o yaml`

---

#### Authenticate via the service account
We can now authenticate via the service account and test that our role and role bindings work as expected.  
First we need to find the JWT signed token, created together with the `jenkins` service account.  

Check the name of the secret via:

`kubectl get serviceaccounts jenkins -o yaml`

The response should be similar to the following (please note the `secrets` entry and the secret name `jenkins-token-928fn`):

```
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: "2020-07-13T15:36:32Z"
  name: jenkins
  namespace: default
  resourceVersion: "367714210"
  selfLink: /api/v1/namespaces/default/serviceaccounts/jenkins
  uid: a0bd6341-c51e-11ea-82db-42010a9a0034
secrets:
- name: jenkins-token-928fn
```

Now extract the secret and assign it to the environment variable `TOKEN`. (remember to replace the secret name `jenkins-token-928fn` with yours)  

```
TOKEN=`kubectl get secret jenkins-token-928fn -o jsonpath='{.data.token}'| base64 --decode`
```

Now we can use kubectl to authenticate to the Kubernetes API via the `jenkins` service account token.  

Verify that you can get the list of pods: `kubectl --token $TOKEN get pods`  
If you don't have any pods, create some via: `kubectl create deployment --image=nginx nginx-app` and try again.  

Now verify that the `jenkins` service account cannot access `secrets` for example via: `kubectl --token $TOKEN get secrets`  

---

#### Cleanup

Now try to cleanup the resources created in this excercise by yourself.