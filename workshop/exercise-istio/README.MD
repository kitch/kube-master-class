# Istio Security

## Prep

Before we get started we'll need a copy of the istio source code. Much of this exercise draws on the Istio examples that can be found here: 

Some of the sample apps are located in the istio git repo. You will need to clone this repo locally:
```
cd ~/kops101/workshop/exercise-istio
git clone https://github.com/istio/istio.git ~/kops101/workshop/exercise-istio/istio
```

Now that we have all the source and samples we'll need, let's get to work.

## Install Istio

Thanks to the magic of IBM Cloud Kubernetes Service you can now simply install a managed Istio deployment by simply running
```
ibmcloud ks clusters
ibmcloud ks cluster-addon-enable istio --cluster <yourcluster>
```

Enable your kubernetes context to point to your cluster
```
ibmcloud ks clusters
$(ibmcloud ks cluster-config --export --cluster <yourcluster>
```

After a few minutes the deployment of Istio will be complete. You can verify this by ensuing that all pods are in running state in
```
kubectl -n istio-system get pods -o wide -w
```
Once all pods reach running state you are ready to go.

## Globally Enable mTLS

This section is largely based upon the documentation and samples found in the official [istio documentation](https://istio.io/docs/tasks/security/authn-policy/#globally-enabling-istio-mutual-tls).

Set up our sample applications
```
kubectl create ns foo
kubectl label ns foo istio-injection=enabled
kubectl apply -f istio/samples/httpbin/httpbin.yaml -n foo
kubectl apply -f istio/samples/sleep/sleep.yaml -n foo
kubectl create ns bar
kubectl label ns bar istio-injection=enabled
kubectl apply -f istio/samples/httpbin/httpbin.yaml -n bar
kubectl apply -f istio/samples/sleep/sleep.yaml -n bar
kubectl create ns legacy
kubectl apply -f istio/samples/httpbin/httpbin.yaml -n legacy
kubectl apply -f istio/samples/sleep/sleep.yaml -n legacy
```
Once we have deployed all of our sample applications we can do a simple verification. This command should return `200` for all checks.
```
for from in "foo" "bar" "legacy"; do for to in "foo" "bar" "legacy"; do kubectl exec $(kubectl get pod -l app=sleep -n ${from} -o jsonpath='{.items..metadata.name}') -c sleep -n ${from} -- curl http://httpbin.${to}:8000/ip -s -o /dev/null -w "sleep.${from} to httpbin.${to}: %{http_code}\n"; done; done
```

### Enable mTLS
Let's now enable global mTLS. This will enable TLS enforcement for the receiving side for all services enabled with Istio sidecars (Envoy).
```
cat <<EOF | kubectl apply -f -
apiVersion: "authentication.istio.io/v1alpha1"
kind: "MeshPolicy"
metadata:
  name: "default"
spec:
  peers:
  - mtls: {}
EOF
```

It may take a few moments for the Istio control plane to roll out all the changes. We will repeat our connection test from earlier. However, this time all the connections will fail with a 503 due to the fact that we have not yet enabled the requesting clients to use TLS.
```
for from in "foo" "bar"; do for to in "foo" "bar"; do kubectl exec $(kubectl get pod -l app=sleep -n ${from} -o jsonpath='{.items..metadata.name}') -c sleep -n ${from} -- curl http://httpbin.${to}:8000/ip -s -o /dev/null -w "sleep.${from} to httpbin.${to}: %{http_code}\n"; done; done
```

We will now set up an Istio DestinationRule which will enable all Istio enabled pods making requests to Istio services will use TLS for connections.
```
cat <<EOF | kubectl apply -f -
apiVersion: "networking.istio.io/v1alpha3"
kind: "DestinationRule"
metadata:
  name: "default"
  namespace: "default"
spec:
  host: "*.local"
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
EOF
```

It may take a few moments for the Istio control plane to roll out all the changes. We will repeat our connection test from earlier. One last time we'll verify our connections. This time we will see all 200s as our DestinationRule and MeshPolicy combined ensure mTLS on both ends of our connection.
```
for from in "foo" "bar"; do for to in "foo" "bar"; do kubectl exec $(kubectl get pod -l app=sleep -n ${from} -o jsonpath='{.items..metadata.name}') -c sleep -n ${from} -- curl http://httpbin.${to}:8000/ip -s -o /dev/null -w "sleep.${from} to httpbin.${to}: %{http_code}\n"; done; done
```

### Test Legacy services to Istio services
Last test. Now we will attempt to connect to services that are in the mesh (foo/bar namespaces) from our legacy namespace which is not in the mesh (legacy namespace). Here connections will fail as expected.
```
for from in "legacy"; do for to in "foo" "bar"; do kubectl exec $(kubectl get pod -l app=sleep -n ${from} -o jsonpath='{.items..metadata.name}') -c sleep -n ${from} -- curl http://httpbin.${to}:8000/ip -s -o /dev/null -w "sleep.${from} to httpbin.${to}: %{http_code}\n"; done; done
```

### Enable Istio services to connect to the kube-apiserver

It's important to set up a DestiationRule to allow Istio services to connect to the kube-apiserver. This will disable tls for connections to the `kubernetes` service. In IKS (and most Kube clusters) standard TLS authentication is used with the kube-apiserver so we disable the Istio mTLS.
```
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
 name: "api-server"
spec:
 host: "kubernetes.default.svc.cluster.local"
 trafficPolicy:
   tls:
     mode: DISABLE
EOF
```

Let's verify that we are able to connect to the kube-apiserver. We should receive a `200` response code.
```
TOKEN=$(kubectl describe secret $(kubectl get secrets | grep default | cut -f1 -d ' ') | grep -E '^token' | cut -f2 -d':' | tr -d '\t')
kubectl exec $(kubectl get pod -l app=sleep -n foo -o jsonpath='{.items..metadata.name}') -c sleep -n foo --  curl https://kubernetes.default/api --header "Authorization: Bearer $TOKEN" --insecure -s -o /dev/null -w "%{http_code}\n"
```

# Cleanup
kubectl delete meshpolicy default
kubectl delete destinationrules default httpbin-legacy api-server
kubectl delete policy default overwrite-example -n foo
kubectl delete policy httpbin -n bar
kubectl delete destinationrules default overwrite-example -n foo
kubectl delete destinationrules httpbin -n bar

# Optional
kubectl delete ns foo bar legacy


# Sleep test and deploy
kubectl apply -f sleep.yaml
export SOURCE_POD=$(kubectl get pod -l app=sleep -o jsonpath='{.items..metadata.name}')
kubectl exec -it $SOURCE_POD -c sleep -- curl -sL -o /dev/null -D - http://edition.cnn.com/politics

cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: cnn
spec:
  hosts:
  - edition.cnn.com
  ports:
  - number: 80
    name: http-port
    protocol: HTTP
  - number: 443
    name: https
    protocol: HTTPS
  resolution: DNS
EOF
kubectl exec -it $SOURCE_POD -c sleep -- curl -sL -o /dev/null -D - http://edition.cnn.com/politics
kubectl apply -f egress.yaml
kubectl apply -f vservice.yaml
kubectl exec -it $SOURCE_POD -c sleep -- curl -sL -o /dev/null -D - http://edition.cnn.com/politics


kubectl delete gateway istio-egressgateway
kubectl delete serviceentry cnn
kubectl delete virtualservice direct-cnn-through-egress-gateway
kubectl delete destinationrule egressgateway-for-cnn


# Globaly Enable mTLS