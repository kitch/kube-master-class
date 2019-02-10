

```
$ kubectl create ns team-a
namespace “team-a” created
```

```
kubectl -n team-a create serviceaccount travis
serviceaccount "travis" created
```

```
$ kubectl create ns team-b
namespace “team-b” created
```

```
kubectl -n team-b create serviceaccount jenkins
serviceaccount "jenkins" created
```

```
jakekit at kitchbook in ~
$ JENKINS_KUBE_TOKEN=`kubectl -n team-a get secret $(kubectl -n serviceids get secret | grep jenkins | awk '{print $1}') -o json | jq -r '.data.token'  | base64 -D`

jakekit at kitchbook in ~
$ TRAVIS_KUBE_TOKEN=`kubectl -n team-b get secret $(kubectl -n serviceids get secret | grep travis | awk '{print $1}') -o json | jq -r '.data.token'  | base64 -D`

jakekit at kitchbook in ~
$ kubectl --server=https://169.61.83.62:31151 --token=$TRAVIS_KUBE_TOKEN get deployments -n squad-a-apps
Error from server (Forbidden): deployments.extensions is forbidden: User "system:serviceaccount:serviceids:travis" cannot list deployments.extensions in the namespace "squad-a-apps"
jakekit at kitchbook in ~

$ kubectl --server=https://169.61.83.62:31151 --token=$JENKINS_KUBE_TOKEN get deployments -n squad-b-apps
Error from server (Forbidden): deployments.extensions is forbidden: User "system:serviceaccount:serviceids:jenkins" cannot list deployments.extensions in the namespace "squad-b-apps"
```