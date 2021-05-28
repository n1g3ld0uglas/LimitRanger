# LimitRanger
Cluster administrators can leverage limit ranges to ensure misbehaving pods, containers, or PersistentVolumeClaims don't consume all available resources.

To use limit ranges, you must first enable the LimitRanger admission controller:
```
ps aux | grep kube-api
```

This admission controller observes the incoming request and ensures that it does not violate any of the limits specified within the 'LimitRange' object.

```
cat limit_range1.yaml
apiVersion: "v1"
kind: "limitRange"
metadata:
  name: "pod-example"
spec:
  limits:
    - type: "Pod"
      max:
        memory: "128Mi"
```

With this limit range object, any pod requesting memory of more than 128 Mi will fail:

```
pods "range-demo" is forbidden maximum memory usage per Pod is 128Mi, but limit is 1073741824
```

When using LimitRanger, malicious pods cannot consume excess resources.

Simply explained, with LimitRanger enabled we can enforce 'default' / 'max' / 'min' limits on storage and computer resources. 
Cluster adminsitrators create a limit range for objects such as pods, containers, and persistentVolumeCliams. 
For any request for object creation or update, the LimitRanger admission controller verifies that the request does not violate any limit ranges. 
If the request violates any limit ranges, a 403 Forbidden response is sent.

Create a namespace where we can apply the LimitRanger.

```
cat limit_range.yaml
apiVersion: "v1"
kind: "LimitRange"
metadata:
  name: limit1
  namespace: demo
spec:
  limits:
  - type: "container"
    max:
      memory: 512Mi
      cpu: 500m
    min:
      memory: 50mi
      cpu: 50m
```

Verify that the LimitRange was applied:

```
kubectl get limitrange -n demo
```

Create a pod that violates the limit range:

```
cat nginx-bad.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-bad
spec:
  containers:
  - name: nginx-bad
    image: nginx-bad
    resources:
      limits:
        memory: "800Mi"
        cpu: "500m"
```

Run a request for the new pod:

```
kubectl apply -f nginx-bad.yaml -n demo
```

It should be denied with the following error:

```
Error from server (Forbidden): error when creating "nginx-bad.yaml": pods "nginx-bad" is forbidden: maximum memory usage per Container is 512Mi, but limit is 800M
```

If a LimitRanger specifies a CPU or memory, all pods and containers should have the CPU or memory requests or limits. LimitRanger works when the request to create or update the object is received bu the API server but not at runtime. If a pod has a violating limit before the limit is applied, it will keep running. Ideally, limits should be applied to the namespace when it is created.


Let's create a new namespace example

```
kubectl create namespace cpu-LimitRange
```

Here's a new configuration file for our LimitRange:

```
cat << EOF > cpu-limit.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-min-max-demo
spec:
  limits:
  - max:
      cpu: "800m"
    min:
      cpu: "200m"
    type: Container
EOF    
```

```
kubectl apply -f cpu-limit.yaml
```

View detailed information about the LimitRange:

```
kubectl get limitrange cpu-min-max-demo --output=yaml --namespace=cpu-LimitRange
```

The output shows the minimum and maximum CPU constraints as expected. 
But notice that even though you didn't specify default values in the configuration file for the LimitRange, they were created automatically.

Now whenever a Container is created in the constraints-cpu-example namespace, Kubernetes performs these steps:

- If the Container does not specify its own CPU request and limit, assign the default CPU request and limit to the Container.
- Verify that the Container specifies a CPU request that is greater than or equal to 200 millicpu.
- Verify that the Container specifies a CPU limit that is less than or equal to 800 millicpu.

