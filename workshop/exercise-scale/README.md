# Scheduling and Autoscaling

## LimitRange
Before we can test auto scaling we'll implement a LimitRange so that all pods created will have default resource requests and limits.

```
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-limit-range
spec:
  limits:
  - default:
      memory: 512Mi
    defaultRequest:
      memory: 256Mi
    type: Container
```
```
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-limit-range
spec:
  limits:
  - default:
      cpu: 20m
    defaultRequest:
      cpu: 10m
    type: Container
```