
# RBAC Exercise

In this part of our lab we'll be learning a little bit about how RBAC works. To accomplish this we'll practice creating a few RBAC policies and reviewing what the imapact is of the policies we create.  IKS also makes available very powerful IBM Cloud IAM integration with RBAC, but here we'll get our hands dirty by implementing it manually.

```
kubectl create ns team-a
kubectl -n team-a create serviceaccount travis
kubectl create ns team-b
kubectl -n team-b create serviceaccount jenkins
```

```
TRAVIS_KUBE_TOKEN=`kubectl -n team-a get secret $(kubectl -n team-a get secret | grep travis | awk '{print $1}') -o json | jq -r '.data.token'  | base64 -D`
```

```
JENKINS_KUBE_TOKEN=`kubectl -n team-b get secret $(kubectl -n team-b get secret | grep jenkins | awk '{print $1}') -o json | jq -r '.data.token'  | base64 -D`
```

```
kubectl --token=$TRAVIS_KUBE_TOKEN get deployments -n team-a
kubectl --token=$JENKINS_KUBE_TOKEN get deployments -n team-b
```

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
  - patch
  - update
  - watch
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



## Cleanup
```
kubectl delete ns team-a team-b
kubectl delete clusterrole cicd-apps
```