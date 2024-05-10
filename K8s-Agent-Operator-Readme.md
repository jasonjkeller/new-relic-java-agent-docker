# Using New Relic Agent Operator With Kubernetes Services
Details the steps to install the [newrelic-agent-operator](https://github.com/newrelic-experimental/newrelic-agent-operator) to inject a New Relic APM agent into a service running in Kubernetes.

## Requirements
* Install helm: https://helm.sh/docs/intro/install/
* Install kubectl: https://kubernetes.io/docs/tasks/tools/
* Install minikube (for local testing): https://minikube.sigs.k8s.io/docs/start/

## Start Minikube Kubernetes Cluster
Ensure that Docker is running and then start up the minikube cluster as follows:
```shell
minikube start
```

## Install cert manager
Add `jetstack` repo:
```shell
helm repo add jetstack https://charts.jetstack.io --force-update
```
Install cert manager in `cert-manager` namespace (which is also created here):
```shell
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.14.4 \
  --set installCRDs=true
```

## Install newrelic-agent-operator
Add `newrelic-agent-operator` repo:
```shell
helm repo add newrelic-agent-operator https://newrelic-experimental.github.io/newrelic-agent-operator
```
Install `newrelic-agent-operator` helm chart in `newrelic` namespace (which is also created here):
```shell
helm upgrade --install newrelic-agent-operator newrelic-agent-operator/newrelic-agent-operator --set licenseKey='<NEW RELIC INGEST LICENSE KEY>' -n newrelic --create-namespace
```

Note that setting `licenseKey` here results in `kubectl` setting a secret named `new_relic_license_key` in the `newrelic` namespace that will be passed along when the agent is injected. 

## Apply instrumentation
Apply instrumentation (and agent config env vars) in `newrelic` namespace:
```shell
kubectl apply -n newrelic -f - <<EOF
apiVersion: newrelic.com/v1alpha1
kind: Instrumentation
metadata:
  labels:
    app.kubernetes.io/name: instrumentation
    app.kubernetes.io/created-by: newrelic-agent-operator
  name: newrelic-instrumentation
spec:
  java:
    image: ghcr.io/newrelic-experimental/newrelic-agent-operator/instrumentation-java:8.10.0
    env:
    - name: NEW_RELIC_APPLICATION_LOGGING_FORWARDING_ENABLED
      value: "false"
  nodejs:
    image: ghcr.io/newrelic-experimental/newrelic-agent-operator/instrumentation-nodejs:11.15.0
  python:
    image: ghcr.io/newrelic-experimental/newrelic-agent-operator/instrumentation-python:9.8.0
  dotnet:
    image: ghcr.io/newrelic-experimental/newrelic-agent-operator/instrumentation-dotnet:10.23.0
  php:
    image: ghcr.io/newrelic-experimental/newrelic-agent-operator/instrumentation-php:10.19.0.9
EOF
```

## Add instrumentation annotation to app
The application running in kubernetes must be configured to add the `instrumentation.newrelic.com/inject-java: "true"` annotation to signal to the operator that it's "opting in" to auto-instrumentation.

The `helm-petclinic-app` in this project adds the annotation in [values.yaml](helm-petclinic-app/values.yaml):

```
podAnnotations:
  'instrumentation.newrelic.com/inject-java': 'true'
```

## Install Java app
Install Java app in the `newrelic` namespace.
```shell
helm install helm-petclinic-app-release helm-petclinic-app -n newrelic
```

After the app is installed, the New Relic APM agent will be auto-injected into it.

Note that the above command installs the app as described in the [README.md](README.md).

A simpler app deployment could be achieved by applying the following instead:
```shell
kubectl apply -n newrelic -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-petclinic
spec:
  selector:
    matchLabels:
      app: spring-petclinic
  replicas: 1
  template:
    metadata:
      labels:
        app: spring-petclinic
      annotations:
        instrumentation.newrelic.com/inject-java: "true"
    spec:
      containers:
        - name: spring-petclinic
          image: ghcr.io/pavolloffay/spring-petclinic:latest
          ports:
            - containerPort: 8080
          env:
          - name: NEW_RELIC_APP_NAME
            value: spring-petclinic-demo
EOF
```

## Validate instrumentation
Validate that the instrumentation applied:
```shell
kubectl describe pod -n newrelic
```

You should see output similar to the following indicating that the auto-injection worked (e.g. for Java `JAVA_TOOL_OPTIONS -javaagent:/newrelic-instrumentation/newrelic-agent.jar`):
```shell
Name:             helm-petclinic-app-release-5c9f94b5b8-k68xp
Namespace:        newrelic
Priority:         0
Service Account:  helm-petclinic-app-release
Node:             minikube/192.168.49.2
Start Time:       Fri, 10 May 2024 10:12:11 -0700
Labels:           app.kubernetes.io/instance=helm-petclinic-app-release
                  app.kubernetes.io/managed-by=Helm
                  app.kubernetes.io/name=helm-petclinic-app
                  helm.sh/chart=helm-petclinic-app-0.1.0
                  pod-template-hash=5c9f94b5b8
Annotations:      instrumentation.newrelic.com/inject-java: true
Status:           Running
IP:               10.244.0.12
IPs:
  IP:           10.244.0.12
Controlled By:  ReplicaSet/helm-petclinic-app-release-5c9f94b5b8
Init Containers:
  newrelic-instrumentation:
    Container ID:  docker://e087c325a4648e92ec885a006a4f4e9dfcce1a0f65b71b1f581bfbfba8cb6e06
    Image:         ghcr.io/newrelic-experimental/newrelic-agent-operator/instrumentation-java:8.10.0
    Image ID:      docker-pullable://ghcr.io/newrelic-experimental/newrelic-agent-operator/instrumentation-java@sha256:22fb51c7f7612883bdb7d78d36675fc34105e2e705a7f2ec25f04b951b3d70df
    Port:          <none>
    Host Port:     <none>
    Command:
      cp
      /newrelic-agent.jar
      /newrelic-instrumentation/newrelic-agent.jar
    State:          Terminated
      Reason:       Completed
      Exit Code:    0
      Started:      Fri, 10 May 2024 10:12:11 -0700
      Finished:     Fri, 10 May 2024 10:12:11 -0700
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /newrelic-instrumentation from newrelic-instrumentation (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-gkgtt (ro)
Containers:
  helm-petclinic-app:
    Container ID:   docker://a6198bc4fcbba3559bdda69dfe234a488025b5a3a7956439caa8c3e17cd7adda
    Image:          jkellernr/new-relic-java-agent-spring-petclinic:petclinic-app
    Image ID:       docker-pullable://jkellernr/new-relic-java-agent-spring-petclinic@sha256:dfce3344f8b7cfb998ae83472a79e3b577461103c8bad55570b41fd176d95259
    Port:           8080/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Fri, 10 May 2024 10:12:13 -0700
    Ready:          True
    Restart Count:  0
    Environment:
      NEW_RELIC_APPLICATION_LOGGING_FORWARDING_ENABLED:  false
      JAVA_TOOL_OPTIONS:                                  -javaagent:/newrelic-instrumentation/newrelic-agent.jar
      NEW_RELIC_APP_NAME:                                helm-petclinic-app-release
      NEW_RELIC_LICENSE_KEY:                             <set to the key 'new_relic_license_key' in secret 'newrelic-key-secret'>  Optional: true
      NEW_RELIC_LABELS:                                  operator:auto-injection
    Mounts:
      /newrelic-instrumentation from newrelic-instrumentation (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-gkgtt (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True
  Initialized                 True
  Ready                       True
  ContainersReady             True
  PodScheduled                True
Volumes:
  kube-api-access-gkgtt:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
  newrelic-instrumentation:
    Type:        EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:
    SizeLimit:   <unset>
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                 node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:          <none>
```

---

## Debugging
None of this is needed for the setup, but it may come in useful if you need to debug.

### View service logs in Kubernetes pod
```shell
kubectl get pods -n newrelic
```  
```shell
kubectl logs <pod_name> -n newrelic
```

### Get a shell into docker container in kubernetes to view Java agent logs  
```shell
kubectl exec -it <pod_name> -n newrelic sh
```
```shell
cd ../newrelic-instrumentation/logs && cat newrelic_agent.log
```

### Check deployment
```shell
kubectl get service -n newrelic
```
```shell
kubectl get deployments -n newrelic
```
```shell
helm list -a -n newrelic
```

### Access service
Get a list of the service names:
```shell
minikube service list
```
Access the service in a web browser:
```shell
minikube service <service_name> -n newrelic
```

## Secrets
None of this is needed for the setup, but it may come in useful if you need to verify that secrets were set correctly.

### Create a Secret
Create a secret for `new_relic_license_key` in the `newrelic` namespace:
```shell
kubectl create secret generic newrelic-key-secret -n newrelic --from-literal=new_relic_license_key=<NEW RELIC INGEST LICENSE KEY>
```

### Get the Secret
Get the Secret in the `newrelic` namespace:
```shell
kubectl get secrets -n newrelic
```

### View the details of the Secret
View the details of the Secret in the `newrelic` namespace:
```shell
kubectl describe secret newrelic-key-secret -n newrelic
```

### View the contents of the Secret
View the contents of the Secret in the `newrelic` namespace:
```shell
kubectl get secret newrelic-key-secret -n newrelic -o jsonpath='{.data}'
```

### Decode the Secret
Decode the Secret after viewing the contents:
```shell
echo <new_relic_license_key output> | base64 --decode
```