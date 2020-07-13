### Authentication hands on

#### Create a namespace
Create a new namespace so that you can carry out all your assignments: `kubectl create namespace YOUR_UNIQUE_NAMESPACE_NAME`

Verify now that the namespace was created: `kubectl get namespaces`

---

#### Create your personal service account
We will create now your service account and assign admin rights.

`kubectl -n YOUR_UNIQUE_NAMESPACE_NAME create serviceaccount YOUR_NAME`

---

#### Cluster-Admin
The cluster already contains a set of pre-defined cluster roles and one of them is: `cluster-admin`.  

>**WARNING** replace the placeholders of `YOUR_UNIQUE_NAMESPACE_NAME` and `YOUR_NAME` with your values!

`kubectl create clusterrolebinding myadmin --serviceaccount=YOUR_UNIQUE_NAMESPACE_NAME:YOUR_NAME --clusterrole=cluster-admin`

---

#### Create a new kubeconfig context

First of all, we need to extract the service account authentication token. (more on this later).

>This extract the secret name:
```bash
TOKENNAME=`kubectl -n YOUR_UNIQUE_NAMESPACE_NAME get serviceaccount/YOUR_NAME -o jsonpath='{.secrets[0].name}'`
```

> This extract the secret value
```bash
TOKEN=`kubectl -n YOUR_UNIQUE_NAMESPACE_NAME get secret $TOKENNAME -o jsonpath='{.data.token}'| base64 --decode`
```

Now we can add a new user definition in our kubeconfig:  

`kubectl config set-credentials YOUR_NAME --token=$TOKEN`

You can set now the current `kubeconfig` context via:  

`kubectl config set-context --current --user=YOUR_NAME`

---

#### Cleanup

Now try to list the other contexts present in your kubeconfig and revert back to the previous default admin context.