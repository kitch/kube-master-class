# Scheduling and Autoscaling

## Setup

Create our hello sandbox namespace
```
kubectl create ns hello
```
## LimitRange
Before we can test auto scaling we'll implement a LimitRange so that all pods created will have default resource requests and limits.

```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-limit-range
  namespace: hello
spec:
  limits:
  - defaultRequest:
      memory: 256Mi
    type: Container
EOF
```

```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-limit-range
  namespace: hello
spec:
  limits:
  - defaultRequest:
      cpu: 10m
    type: Container
EOF
```

## Test application
Now let's deploy our simple hello test application and service

```
kubectl -n hello run hello-app --image=kitch/hello-app:1.0 --replicas 3
```

Now let's see what our new pods look like

```
kubectl -n hello get pod -l run=hello-app -o yaml
```

You'll notice that these pods have had requests applied to them for us by the limit range, even though the deployment does not have any defined.

```
kubectl -n hello get pods -o yaml | grep -A 1 -B 3 cpu:
```
Next lets expose our new deployment as a service.
```
kubectl -n hello expose deploy hello-app --name hello-svc --port=80 --target-port=8080
```

## Enable autoscaling

Basic horizontal pod autoscaling is very simple and straightforward

```
kubectl -n hello autoscale deploy hello-app --min=1 --max=10 --cpu-percent=80
```

This will enable autoscaling for our hello-app deployment.

```
kubectl -n hello get hpa
```

Initially the HPA may not have any data for the current CPU usage, in a few minutes we should see data start to roll in. You can also check via `kubectl top pods`. Once we have some metrics data for our HPA let's move on to the next step.

## Generate workload

We can use this simple command to drive load against our test application
```
kubectl -n hello run hello-load --image=alpine --command -- sh -c 'apk update && apk add curl && while true; do curl hello-svc; done'
```

Once we start up our load generator we can use a watch to _watch_ what is happening with our autoscaler

```
kubectl -n hello get hpa -w
```

In a few minutes we'll see CPU start to spike and the HPA will scale up our application to meet demand based on our scaling policy. Once the app has scale up successfully we can delete our load generator and watch it scale back down.

```
kubectl -n hello delete deploy hello-load
kubectl get hpa -w
```



## Cleanup

```
kubectl delete ns hello
```
