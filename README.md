# custom-kubernetes-operator


### Prerequisite

- Docker
- Kubernetes
- Go 1.19+ (support go module)

### Checkout

First, clone the application to any directory as long as not in GOPATH (since this project use Go Module, so clonning in GOPATH is not recommended)

```
$ git clone git@github.com:ligert/custom-kubernetes-operator.git 

$ cd custom-kubernetes-operator
```

### Unit Tests
To run unit tests, run this command
```
$ make test
```

### Deployment

To run the deployment, run this command

```
$ make deploy
```

The output should be something similar to this
```
test -s /home/custom-kubernetes-operator/bin/controller-gen || GOBIN=/home/custom-kubernetes-operator/bin go install sigs.k8s.io/controller-tools/cmd/controller-gen@v0.10.0
/home/custom-kubernetes-operator/bin/controller-gen rbac:roleName=manager-role crd webhook paths="./..." output:crd:artifacts:config=config/crd/bases
test -s /home/custom-kubernetes-operator/bin/kustomize || { curl -Ss "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash -s -- 3.8.7 /home/custom-kubernetes-operator/bin; }
cd config/manager && /home/custom-kubernetes-operator/bin/kustomize edit set image controller=ligert/custom-kubernetes-operator:latest
/home/custom-kubernetes-operator/bin/kustomize build config/default | kubectl apply -f -
namespace/custom-kubernetes-operator-system created
customresourcedefinition.apiextensions.k8s.io/dummies.interview.com created
serviceaccount/custom-kubernetes-operator-controller-manager created
role.rbac.authorization.k8s.io/custom-kubernetes-operator-leader-election-role created
clusterrole.rbac.authorization.k8s.io/custom-kubernetes-operator-manager-role created
clusterrole.rbac.authorization.k8s.io/custom-kubernetes-operator-metrics-reader created
clusterrole.rbac.authorization.k8s.io/custom-kubernetes-operator-proxy-role created
rolebinding.rbac.authorization.k8s.io/custom-kubernetes-operator-leader-election-rolebinding created
clusterrolebinding.rbac.authorization.k8s.io/custom-kubernetes-operator-manager-rolebinding created
clusterrolebinding.rbac.authorization.k8s.io/custom-kubernetes-operator-proxy-rolebinding created
service/custom-kubernetes-operator-controller-manager-metrics-service created
deployment.apps/custom-kubernetes-operator-controller-manager created
```

To check that the deployment worked correctly, you get the deployments and the pods
```
$ kubectl get deploy --all-namespaces

NAMESPACE                           NAME                                            READY   UP-TO-DATE   AVAILABLE   AGE
custom-kubernetes-operator-system   custom-kubernetes-operator-controller-manager   1/1     1            1           88s

$ kubectl get pods --all-namespaces

NAMESPACE                           NAME                                                            READY   STATUS    RESTARTS         AGE
custom-kubernetes-operator-system   custom-kubernetes-operator-controller-manager-9f7fc8fbb-wg5zs   2/2     Running   0                2m10s

```

### Creating the dummy object

To create the Dummy object, apply the sample yaml. A new pod running nginx should appear
```
$ kubectl apply -f config/samples/_v1alpha1_dummy.yaml 

$ kubectl get pods --all-namespaces

NAMESPACE                           NAME                                                            READY   STATUS    RESTARTS         AGE
custom-kubernetes-operator-system   custom-kubernetes-operator-controller-manager-9f7fc8fbb-wg5zs   2/2     Running   0                2m46s
default                             dummy-sample-749f6d77cc-mfrvg                                   1/1     Running   0                6s
```

### Check Nginx pod

We can verifiy that nginx is running in the dummy-sample-* pod
```
kubectl logs dummy-sample-749f6d77cc-mfrvg (example)

2022/10/19 04:36:21 [notice] 1#1: using the "epoll" event method
2022/10/19 04:36:21 [notice] 1#1: nginx/1.23.1
2022/10/19 04:36:21 [notice] 1#1: built by gcc 10.2.1 20210110 (Debian 10.2.1-6) 
2022/10/19 04:36:21 [notice] 1#1: OS: Linux 5.10.16.3-microsoft-standard-WSL2
2022/10/19 04:36:21 [notice] 1#1: getrlimit(RLIMIT_NOFILE): 1048576:1048576
2022/10/19 04:36:21 [notice] 1#1: start worker processes
2022/10/19 04:36:21 [notice] 1#1: start worker process 28
2022/10/19 04:36:21 [notice] 1#1: start worker process 29
```

### Verify yaml

We can check the dummy yaml to verify that the spec has been copied to the status.echoSpec, and that the podStatus is correctly `Running`
```
$ kubectl get dummies.interview.com/dummy-sample -o yaml

apiVersion: interview.com/v1alpha1
kind: Dummy
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"interview.com/v1alpha1","kind":"Dummy","metadata":{"annotations":{},"name":"dummy-sample","namespace":"default"},"spec":{"message":"I'm a dummy"}}
  creationTimestamp: "2022-10-19T04:36:20Z"
  generation: 1
  name: dummy-sample
  namespace: default
  resourceVersion: "117286"
  uid: 3eb52d19-9ca1-4262-a3d7-0d416436ccd8
spec:
  message: I'm a dummy
status:
  echoSpec: I'm a dummy
  podStatus: Running
```


### Shutting down
To stop the deployment, run these commands
After executing the delete, the nginx pod associated with the dummy object will be terminated too
``` 
kubectl delete -f config/samples/_v1alpha1_dummy.yaml

NAMESPACE                           NAME                                                            READY   STATUS    RESTARTS       AGE
custom-kubernetes-operator-system   custom-kubernetes-operator-controller-manager-9f7fc8fbb-f2949   2/2     Running   0              9m3s


make undeploy
```



### Test Results

We create the deployment and verifiy that the controller is created
```
make deploy
```
![alt text](https://github.com/ligertsicat/custom-kubernetes-operator/blob/master/blob/screenshot1.PNG?raw=true)

After that we can apply the dummy yaml
```
kubectl apply -f config/samples/_v1alpha1_dummy.yaml 
```
An nginx pod associated with the Dummy API will be created
![alt text](https://github.com/ligertsicat/custom-kubernetes-operator/blob/master/blob/screenshot2.PNG?raw=true)

The dummy name, namespace, and message are logged in the controller
![alt text](https://github.com/ligertsicat/custom-kubernetes-operator/blob/master/blob/screenshot3.PNG?raw=true)

The pod is running nginx
![alt text](https://github.com/ligertsicat/custom-kubernetes-operator/blob/master/blob/screenshot4.PNG?raw=true)

The yaml status.echoSpec and status.podStatus are correctly set to "I'm a dummy" and "Running"
![alt text](https://github.com/ligertsicat/custom-kubernetes-operator/blob/master/blob/screenshot5.PNG?raw=true)

Deleting the Dummy API results in the nginx pod being also terminated
![alt text](https://github.com/ligertsicat/custom-kubernetes-operator/blob/master/blob/screenshot6.PNG?raw=true)

```
make test
```
Unit tests are successful
![alt text](https://github.com/ligertsicat/custom-kubernetes-operator/blob/master/screenshot6.PNG?raw=true)

