### Quotas
An administrator can limit the amount of resources a namespace can use by levereging the resource quotas.  
This is particularly important if there are concerns regarding the total amount of resources a cluster has and a fair distribution of such resources. 

#### Namespace quotas 
1. An admin uses `ResourceQuota` for the namespaces.
2. Users of the namespace create resources in it and Kubernetes keeps track of the usage and makes sure the limits are honoured.
3. If creating a resource in the namespace exceed the quota, the request to schedule the resource will fail with a status code of `403 FORBIDDEN`
4. **IMPORTANT** if the quota is enabled in a namespace for compute resources like `CPU` and `MEMORY` then users of the namespace need to set the `Requests` or `Limits` in theor YAML files.

---
 
#### Set a ResourceQuota

1. Create a new namespace.
2. Create a local file called `quotas.yaml` and copy the following content:  
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-resources
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
    requests.nvidia.com/gpu: 1
```

3. Apply the quota to the namespace: `kubectl create -f ./compute-resources.yaml --namespace=myspace` (change the `myspace` with the name of the namespace you created)

---

#### Create resources

Try now to create a resource in this namespace and check whether it gets created:  

1. Run an Nginx deployment: `kubectl run --namespace myspace --image=nginx nginx-app`
2. Verify an error is thrown because we do not specify either the `Requests` and/or `Limits` for the resource.
This is an example of the expected error:  
```bash
15s         Warning   FailedCreate        replicaset/nginx-app-78f96d5496   Error creating: pods "nginx-app-78f96d5496-hxcb9" is forbidden: failed quota: compute-resources: must specify limits.cpu,limits.memory,requests.cpu,requests.memory
```

---

#### Fix the deployment

Kubernetes will keep trying to deploy Nginx in this namespace.  
We have two ways to fix the deployment:  
1. Modify on the fly the Nginx deployment
2. Use `LimitRange` which will allow to set default limits for every resource deployed in the namespace.

##### On the fly fix
1. Edit the deployment on the fly via: `kubectl edit deployment nginx-app -n myspace`
2. Adjust it as follow:  
```yaml
    spec:
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: nginx-app
        resources:
          limits:
            cpu: "1"
          requests:
            cpu: 500m
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
```
3. Verify that the deployment succeeded: `kubectl get events -n myspace`

##### LimitRange fix

1. Delete the current deployment via: 
2. Create the LimitRange:  
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: limit-range
spec:
  limits:
  - default:
      cpu: 1
      memory: 512Mi
    defaultRequest:
      cpu: 0.5
      memory: 256Mi
    type: Container
```
3. Verify that the LimitRange was created: `kubectl get limitrange -n myspace`
4. Request an Nginx deployment: `kubectl run --namespace myspace --image=nginx nginx-app`
3. Verify that the deployment succeeded: `kubectl get events -n myspace`

---

#### Cleanup

Remove the namespace you created. It will automatically delete also the `LimitRange`  
