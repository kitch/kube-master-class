
# RBAC Exercise

In this part of our lab we'll be learning a little bit about how RBAC works. To accomplish this we'll practice creating a few RBAC policies and reviewing what the imapact is of the policies we create.  IKS also makes available very powerful IBM Cloud IAM integration with RBAC, but here we'll get our hands dirty by implementing it manually.


## Setup service accounts

For this exercise we'll create a couple of new namespacesa and service accounts to allow us to easily demonstrate the RBAC functionality

```
kubectl create ns team-a
kubectl -n team-a create serviceaccount travis
kubectl create ns team-b
kubectl -n team-b create serviceaccount jenkins
```

Here we'll set a couple of environment variables to let us "impersonate" the service accounts by using their tokens for auth.
```
TRAVIS_KUBE_TOKEN=`kubectl -n team-a get secret $(kubectl -n team-a get secret | grep travis | awk '{print $1}') -o json | jq -r '.data.token'  | base64 -d`
```

```
JENKINS_KUBE_TOKEN=`kubectl -n team-b get secret $(kubectl -n team-b get secret | grep jenkins | awk '{print $1}') -o json | jq -r '.data.token'  | base64 -d`
```

Now that we have authentication, lets try out our new service accounts
```
kubectl --token=$TRAVIS_KUBE_TOKEN get deployments -n team-a
kubectl --token=$JENKINS_KUBE_TOKEN get deployments -n team-b
```

You'll find that both requests will fail as we have not provided these accounts with any RBAC authorization yet.

## Create our RBAC

Now we'll create a ClusterRole as well as some bindings in order to authorize our service accounts to manage deployments.

```
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: cicd-apps
rules:
- apiGroups:
  - apps
  - extensions
  resources:
  - deployments
  - replicasets
  verbs:
  - create
  - delete
  - deletecollection
  - get
  - list

EOF
```


```
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: travis-apps
  namespace: team-a
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cicd-apps
subjects:
- kind: ServiceAccount
  name: travis
  namespace: team-a
EOF
```

```
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: jenkins-apps
  namespace: team-b
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cicd-apps
subjects:
- kind: ServiceAccount
  name: jenkins
  namespace: team-b
EOF
```

Let's try pulling those deployments again.
```
kubectl --token=$TRAVIS_KUBE_TOKEN get deployments -n team-a
kubectl --token=$JENKINS_KUBE_TOKEN get deployments -n team-b
```

## Using our accounts

Now we can test deploying some apps just as our pipeline tools would do so in real life.

```
kubectl --token=$TRAVIS_KUBE_TOKEN -n team-a run hello-app --image=kitch/hello-app:1.0 --replicas 3
kubectl --token=$JENKINS_KUBE_TOKEN -n team-b run hello-app --image=kitch/hello-app:1.0 --replicas 3
```

Let's see how our new deployments are doing

```
kubectl --token=$TRAVIS_KUBE_TOKEN -n team-a get deployments
kubectl --token=$JENKINS_KUBE_TOKEN -n team-b get deployments
```

Let's see if we can get some more details on the new pods

```
kubectl --token=$TRAVIS_KUBE_TOKEN -n team-a get pods
kubectl --token=$JENKINS_KUBE_TOKEN -n team-b get pods
```

This will fail as we have not provided our users authorization for management of pods. These are meant for pipeline tools, which really only need to manage the deployment artifacts, not the runtime components.

## Cleanup
```
kubectl delete ns team-a team-b
kubectl delete clusterrole cicd-apps
```
