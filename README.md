# Dockerized Springboot Petclinic Service With New Relic Java Agent

Dockerized version of the [SpringBoot PetClinic service](https://github.com/spring-projects/spring-petclinic) with the [New Relic Java Agent](https://docs.newrelic.com/docs/apm/agents/java-agent/getting-started/introduction-new-relic-java/).

## Configure the Java agent

Before running the container you must modify the following environment variables in the Dockerfile to configure where the Java agent reports.

These two define a unique entity that is associated with a specific APM account and New Relic environment:
* Set the APM entity name: `ENV NEW_RELIC_APP_NAME=JavaPetClinic`
* License key for account: `ENV NEW_RELIC_LICENSE_KEY='<license_key>'`

Set the following based on which New Relic environment the APM account is associated with:
* US Production
    ```
    ENV NEW_RELIC_HOST=collector.newrelic.com
    ENV NEW_RELIC_API_HOST=rpm.newrelic.com
    ENV NEW_RELIC_METRIC_INGEST_URI=https://metric-api.newrelic.com/metric/v1
    ENV NEW_RELIC_EVENT_INGEST_URI=https://insights-collector.newrelic.com/v1/accounts/events
    ```
* EU Production
    ```
    ENV NEW_RELIC_HOST=collector.eu01.nr-data.net
    ENV NEW_RELIC_API_HOST=api.eu.newrelic.com
    ENV NEW_RELIC_METRIC_INGEST_URI=https://metric-api.eu.newrelic.com/metric/v1
    ENV NEW_RELIC_EVENT_INGEST_URI=https://insights-collector.eu01.nr-data.net/v1/accounts/events
    ```
* US Staging
    ```
    ENV NEW_RELIC_HOST=staging-collector.newrelic.com
    ENV NEW_RELIC_API_HOST=staging.newrelic.com
    ENV NEW_RELIC_METRIC_INGEST_URI=https://staging-metric-api.newrelic.com/metric/v1
    ENV NEW_RELIC_EVENT_INGEST_URI=https://staging-insights-collector.newrelic.com/v1/accounts/events
    ```

(OPTIONAL) Enable JFR monitoring for enhanced JVM details:
* `ENV NEW_RELIC_JFR_ENABLED=true`

## Building/Running Dockerized Petclinic Service

### Option 1: Docker Compose

Build and run:
`docker-compose up -d`

Force a rebuild:
`docker-compose build`

Stop:
`docker-compose down`

### Option 2: Docker Build/Run

Build Docker Image:
`docker build --tag petclinic-app .`

Run Docker Container:
`docker run -p 8080:8080 petclinic-app`

Stop Docker Container:
`docker ps`
`docker stop <CONTAINER ID>`

## Make a Request to the Petclinic Service

By default, the Petclinic Service will be accessible at: http://localhost:8080

Example `curl` request:
`curl --request GET --url http://localhost:8080/vets --header 'content-type: application/json'`

## Docker Hub

**WARNING**: Publishing to Docker Hub is only necessary if you are deploying to a Kubernetes cluster using the Helm chart instructions below. When publishing to Docker Hub do **NOT** configure any secrets (such as `NEW_RELIC_LICENSE_KEY`) as part of the Docker image. These secrets will instead be configured by the Helm charts and passed through to the Docker container. 

Steps to publish image to Docker Hub after it has been built.

1. Docker Login: `docker login`
2. Docker Tag: `docker tag petclinic-app jkellernr/new-relic-java-agent-spring-petclinic:petclinic-app` 
3. Docker Push: `docker push jkellernr/new-relic-java-agent-spring-petclinic:petclinic-app`
4. Docker Pull: `docker pull jkellernr/new-relic-java-agent-spring-petclinic:petclinic-app`

## Helm Charts For Publishing To Kubernetes

A helm chart has already been created (i.e. `helm create <chart_name>`) and configured for this project. All helm chart files can be found in the `helm-petclinic-app` directory.

### Requirements

* Install helm: https://helm.sh/docs/intro/install/
* Install kubectl: https://kubernetes.io/docs/tasks/tools/
* Install minikube (for local testing): https://minikube.sigs.k8s.io/docs/start/

### Start Minikube Kubernetes Cluster

Ensure that Docker is running and then start up the minikube cluster as follows: 

`minikube start`

### Configure Helm Chart To Use The Docker File For Petclinic Service

A publicly available Docker file of the SpringBoot Petclinic service with the New Relic Java agent has been published to Dockerhub repository at: `jkellernr/new-relic-java-agent-spring-petclinic:petclinic-app`

If you wish to use a different Docker image you can simply change the repository in the `helm-petclinic-app/values.yaml` as shown below:  
```yaml
image:
  repository: jkellernr/new-relic-java-agent-spring-petclinic:petclinic-app
```

### Install Helm Chart

By default, the Docker image is configured such that the Java agent will report data to a US Production New Relic APM environment, at a minimum you must set the `new_relic_license_key` for your US Production APM account when installing the helm chart.

#### Helm Chart Install Dry Run Debugging 

It is advisable to do a dry run (i.e. `helm install <chart_release_name> --dry-run --debug <chart_name> -n newrelic`) to debug before installing the chart:  
```shell
helm install helm-petclinic-app-release --dry-run --debug helm-petclinic-app -n newrelic
```

#### Helm Chart Install

Install (i.e. `helm install <chart_release_name> <chart_name> -n newrelic`) the helm chart:  
```shell
helm install helm-petclinic-app-release helm-petclinic-app -n newrelic
```

#### Inspect Generated Helm Chart

Check the config generated for the helm chart after it is installed:

```shell
kubectl describe service helm-petclinic-app-release -n newrelic
kubectl describe pod helm-petclinic-app-release -n newrelic
```

#### Expose Kubernetes Application On Localhost

After the Helm chart has been installed successfully you'll see some commands like the following logged to the console. Execute those commands to make the application running in the Kubernetes cluster accessible on localhost of your dev machine.

```
NOTES:
1. Get the application URL by running these commands:
  export POD_NAME=$(kubectl get pods --namespace newrelic -l "app.kubernetes.io/name=helm-petclinic-app,app.kubernetes.io/instance=helm-petclinic-app-release" -o jsonpath="{.items[0].metadata.name}")
  export CONTAINER_PORT=$(kubectl get pod --namespace newrelic $POD_NAME -o jsonpath="{.spec.containers[0].ports[0].containerPort}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl --namespace newrelic port-forward $POD_NAME 8080:$CONTAINER_PORT
```

Alternatively, if a service is already running you can get a list of the service names via:

`minikube service list`

And access the service in a web browser via:

`minikube service <service_name> -n newrelic`

#### Making Changes To The Helm Chart Install

You can uninstall (i.e. `helm uninstall <chart_release_name> -n newrelic`) the helm chart with the following command and then install it again with the updated settings:  
```shell
helm uninstall helm-petclinic-app-release -n newrelic
```

#### Verifying Helm Chart Deployment

Check deployment: `helm list -a -n newrelic` and `kubectl get deployments -n newrelic`  
Check where deployment is running: `kubectl get service -n newrelic`

#### Debugging

View PetClinic service logs in Kubernetes:  
`kubectl get pods -n newrelic`  
`kubectl logs <pod_name> -n newrelic`  

Get shell into docker container in kubernetes to view Java agent logs:  
`kubectl exec -it <pod_name> -n newrelic sh`  
`cd ../newrelic-instrumentation/logs && cat newrelic_agent.log`  
