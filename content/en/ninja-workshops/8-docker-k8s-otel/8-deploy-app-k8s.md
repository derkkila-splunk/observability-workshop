---
title: Deploy Application to K8s
linkTitle: 8. Deploy Application to K8s
weight: 8
time: 15 minutes
---

## Update the Dockerfile

With Kubernetes, environment variables are typically managed in the `.yaml` manifest files rather 
than baking them into the Docker image.  So let's remove the following two environment variables from the Dockerfile:

``` bash
vi /home/splunk/workshop/docker-k8s-otel/helloworld/Dockerfile
```
Then remove the following two environment variables: 

``` dockerfile
ENV OTEL_SERVICE_NAME=helloworld
ENV OTEL_RESOURCE_ATTRIBUTES='deployment.environment=otel-$INSTANCE'
```
> To save your changes in vi, press the `esc` key to enter command mode, then type `wq!` followed by pressing the `enter/return` key.

## Build a new Docker Image 

Let's build a new Docker image that excludes the environment variables:

``` bash
cd /home/splunk/workshop/docker-k8s-otel/helloworld 

docker build -t helloworld:1.2 .
```

> Note: we've used a different version (1.2) to distinguish the image from our earlier version.
> To clean up the older versions, run the following command to get the container id:
> ``` bash
> docker ps -a
> ```
> Then run the following command to delete the container:
> ``` bash
> docker rm <old container id> --force
> ```
> Now we can get the container image id:
> ``` bash
> docker images | grep 1.1
> ```
> Finally, we can run the following command to delete the old image:
> ``` bash
> docker image rm <old image id>
> ```

## Import the Docker Image to Kubernetes

Normally we’d push our Docker image to a repository such as Docker Hub.
But for this session, we’ll use a workaround to import it to k3s directly. 

``` bash
cd /home/splunk

# Export the image from docker
docker save --output helloworld.tar helloworld:1.2

# Import the image into k3s
sudo k3s ctr images import helloworld.tar
```

## Deploy the .NET Application

> Hint: To enter edit mode in vi, press the 'i' key. To save changes, press the `esc` key to enter command mode, then type `wq!` followed by pressing the `enter/return` key. 

To deploy our .NET application to K8s, let's create a file named `deployment.yaml` in `/home/splunk`:

``` bash
vi /home/splunk/deployment.yaml
```

And paste in the following:

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: helloworld
spec:
  selector:
    matchLabels:
      app: helloworld
  replicas: 1
  template:
    metadata:
      labels:
        app: helloworld
    spec:
      containers:
        - name: helloworld
          image: docker.io/library/helloworld:1.2
          imagePullPolicy: Never
          ports:
            - containerPort: 8080
          env:
            - name: PORT
              value: "8080"
```
> [!tip]- What is a Deployment in Kubernetes?
> The deployment.yaml file is a kubernetes config file that is used to define a deployment resource. This file is the cornerstone of managing applications in Kubernetes! The deployment config defines the deployment’s ***desired state*** and Kubernetes then ensures the ***actual*** state matches it. This allows application pods to self-heal and also allows for easy updates or roll backs to applications.

Then, create a second file in the same directory named `service.yaml`: 

``` bash
vi service.yaml
```

And paste in the following: 

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: helloworld
  labels:
    app: helloworld
spec:
  type: ClusterIP
  selector:
    app: helloworld
  ports:
    - port: 8080
      protocol: TCP
```

> [!tip]- What is a Service in Kubernetes?
> A Service in Kubernetes is an abstraction layer, working like a middleman, giving you a fixed IP address or DNS name to access your Pods, which stays the same, even if Pods are added, removed, or replaced over time. 

We can then use these manifest files to deploy our application: 

{{< tabs >}}
{{% tab title="Script" %}}

``` bash
# create the deployment
kubectl apply -f deployment.yaml

# create the service
kubectl apply -f service.yaml
```

{{% /tab %}}
{{% tab title="Example Output" %}}

``` bash
deployment.apps/helloworld created
service/helloworld created
```

{{% /tab %}}
{{< /tabs >}}

## Test the Application

To access our application, we need to first get the IP address: 

{{< tabs >}}
{{% tab title="Script" %}}

``` bash
kubectl describe svc helloworld | grep IP:
```

{{% /tab %}}
{{% tab title="Example Output" %}}

``` bash
IP:                10.43.102.103
```

{{% /tab %}}
{{< /tabs >}}

Then we can access the application by using the Cluster IP that was returned 
from the previous command.  For example: 

``` bash
curl http://10.43.102.103:8080/hello/Kubernetes
```

## Configure OpenTelemetry 

OpenTelemetry instrumentation was already baked into the Docker image.  But we need to set a few 
environment variables to tell it where to send the data. 

Add the following to `deployment.yaml` file you created earlier: 

> **IMPORTANT** replace `$INSTANCE` in your Dockerfile with your instance name,
> which can be determined by running `echo $INSTANCE`.

``` yaml
          env:
            - name: PORT
              value: "8080"
            - name: NODE_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.hostIP
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: "http://$(NODE_IP):4318"
            - name: OTEL_SERVICE_NAME
              value: "helloworld"
            - name: OTEL_RESOURCE_ATTRIBUTES 
              value: "deployment.environment=otel-$INSTANCE" 
```

Apply the changes with: 

{{< tabs >}}
{{% tab title="Script" %}}

``` bash
kubectl apply -f deployment.yaml
```
{{% /tab %}}
{{% tab title="Example Output" %}}
``` bash
deployment.apps/helloworld configured
```
{{% /tab %}}
{{< /tabs >}}

Then use `curl` to generate some traffic. 

After a minute or so, you should see traces flowing in the o11y cloud. 