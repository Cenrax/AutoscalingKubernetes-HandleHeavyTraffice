# AutoscalingKubernetes-HandleHeavyTraffic

## Problem Scenario
Your organization built an app in Kubernetes and now it’s getting popular! You went from just a few visitors to hundreds (and sometimes thousands) per day. But there’s a problem…the increased traffic is bottlenecking, causing latency and timeouts for your customers. If you can’t improve the experience, people will stop using the app.

You – the brave Kubernetes engineer – have a solution. You deploy an Ingress controller to route the traffic and set up an autoscale policy so that the number of Ingress controller pods instantly expands and contracts to match traffic fluctuations. Now, your Ingress controller pods seamlessly handle the traffic surges - "Goodbye, latency!", and when traffic decreases the pods scale down to conserve resources - "Hello, cost savings!" - well done, you.


## Description and Technology Used


## Solution


### Part 1
We use a YAML file to create a Deployment with a single replica and a service.
We will Deploy the Podinfo App
```
kubectl apply -f 1-deployment.yaml
```
You should get this output:
```
deployment.apps/podinfo created
service/podinfo created
```
We can confirm using the following command

```
kubectl get pods
```
It will retrieve your app - "podinfo" - and should show 1/1 and "running". You won't see any other pods since this is the first item deployed in your cluster.

```
NAME                       READY   STATUS    RESTARTS   AGE
podinfo-5d76864686-rd2s5   1/1     Running   0     

```
If everything has run succesfully we will be able to see the app in our localhost

### Part 2 : Use NGINX Ingress Controller to expose the app

![image](https://user-images.githubusercontent.com/43017632/158034718-8e150a34-513b-4072-aa31-472b09e0d2cd.png)


#### Step 1: Install NGINX Ingress Controller using Helm
The fastest way to install NGINX Ingress Controller is with Helm, which is already installed on this host.
Begin by adding the NGINX repository to Helm:

```
helm repo add nginx-stable https://helm.nginx.com/stable
```
Next, download and install NGINX Ingress Controller in your cluster. We're using the open source version maintained by F5 NGINX.

```
  helm install main nginx-stable/nginx-ingress \
  --set controller.watchIngressWithoutClass=true
  --set controller.service.type=NodePort \
  --set controller.service.httpPort.nodePort=30005
```

If successful, you should see something similar to this output:
```
NAME: main
LAST DEPLOYED: Tue Feb 22 19:49:17 2022
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
The NGINX Ingress Controller has been installed.
```
We can verify whether the traffic was installed correctly by 
```
kubectl get pods

```
We will see a result like this


<img width="506" alt="image" src="https://user-images.githubusercontent.com/43017632/158030864-ad4eeb25-fef4-4ac1-8788-e8571b50cf5b.png">

#### Step 2: Ingress manifest to route traffic to your app

For this we use 2-ingress.yaml file and run it using the following command

```
kubectl apply -f 2-ingress.yaml
```
We should get a output like this:

<img width="325" alt="image" src="https://user-images.githubusercontent.com/43017632/158031002-c44d6dbb-4670-4da6-b7f0-a880db34b3f7.png">

### Part 3: Visualize and Generate Traffic

This part will be focused on the following parts:

- Use Prometheus to get visibility into NGINX Ingress Controller performance
- Use Locust to simulate a traffic surge
- Observe the impact of increased traffic on NGINX Ingress Controller performance

#### Step 1: Exploring Metrics

An Ingress controller is a regular pod that bundles a reverse proxy (NGINX) with some code that integrates with Kubernetes. If our app will receive a lot of traffic, we will want to scale the number of NGINX Ingress Controller pods and increase the replica count. To do this, we need metrics.

NGINX Ingress Controller [exposes multiple metrics](https://github.com/nginxinc/nginx-prometheus-exporter#exported-metrics). There are eight metrics for the NGINX Ingress Controller you're using in this lab (based on NGINX Open Source) and 80+ metrics for the option based on NGINX Plus.

First we get the IP address of the NGINX Ingress Controller pod. 

```
kubectl get pods -o wide
```
We get results like this in the screen
<img width="610" alt="image" src="https://user-images.githubusercontent.com/43017632/158031245-b2e7e03d-e5f5-4851-87f4-e1b82871200b.png">

Now we create a ***temporary*** pod
```
kubectl run -ti --rm=true busybox --image=busybox
```
It opens a shell like this, don't close this:

<img width="505" alt="image" src="https://user-images.githubusercontent.com/43017632/158031407-f621c2ed-a3f4-4189-96b7-1e43b944de43.png">


#### Step 2: View Available Metrics
Use the following command to retrieve a list of the available metrics (note that it includes the IP address you found earlier):

```
wget -qO- 172.17.0.4:9113/metrics
```

We will get a list of available metrics
For scaling purposes, ***nginx_connections_active*** is ideal because it keeps track of the number of requests being actively processed, which will help you identify when a pod needs to scale.

Type exit at the command prompt of the temporary pod to get back to the Kubernetes server.

#### Step 3: Install Prometheus

To trigger autoscaling based on nginx_connections_active, you need two tools:

A mechanism to scrape the metrics - we'll use Prometheus
A tool to store and expose the metrics so that Kubernetes can use them - we'll use KEDA.
Prometheus is a popular open source project of the Cloud Native Computing Foundation (CNCF) for monitoring and alerting. NGINX Ingress Controller offers Prometheus metrics that are useful for visualization and troubleshooting.

Add the Prometheus repository to Helm using this command:
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```
Install prometheus using the following command

```
helm install prometheus prometheus-community/prometheus --set server.service.type=NodePort --set server.service.nodePort=30010
```
  
We can verify using kubectl get pods and we should get something like this

<img width="532" alt="image" src="https://user-images.githubusercontent.com/43017632/158031589-11472f61-cfae-4908-88dd-ad20d7417709.png">

#### Step 3: Query Prometheus

We can open the prometheus port in our tab and select nginx_ingress_nginx_connections_active in the search bar

<img width="538" alt="image" src="https://user-images.githubusercontent.com/43017632/158031726-e704f9f5-1788-456d-9322-c80d3801f945.png">

### Step 4: Installation of Locust

We will use [Locust](https://locust.io/) to simulate a traffic surge that we can detect with the Prometheus dashboard.

We will create a YAML file (3-locust.yaml) to create a pod for the load generator.
Locust reads the following locustfile.py, which is stored in a ConfigMap. The script issues a request to the pod with the correct headers.
```
from locust import HttpUser, task, between

class QuickstartUser(HttpUser):
    wait_time = between(0.7, 1.3)

    @task
    def hello_world(self):
        self.client.get("/", headers={"Host": "example.com"})
```
 
After deploying the pod successfully, we should see this on the local host (ui of the locust)

<img width="955" alt="image" src="https://user-images.githubusercontent.com/43017632/158035026-17f5426b-ce2a-4644-9500-8c1384b7f57e.png">

To simulate a traffic surge, enter the following details into Locust:

Number of users: 1000
Spawn rate: 10
Host: http://main-nginx-ingress
Click "Start swarming" and observe the traffic reaching NGINX Ingress Controller.

<img width="600" alt="image" src="https://user-images.githubusercontent.com/43017632/158035096-00ceec5a-b4ab-4f5c-8672-3adcafc74dd4.png">


<img width="600" alt="image" src="https://user-images.githubusercontent.com/43017632/158035186-ba225d5d-06ad-46fc-9ad3-dc4cc479ca44.png">

Now that you have traffic routing through NGINX Ingress Controller to your Podinfo app, we can return to Prometheus to see how the Ingress controller responds.
As a vast amount of connections are issued, the single Ingress controller pod struggles to process the increased traffic without latency.
Through observing where performance degrades, you might notice that 100 active connections is a tipping point for latency issues. This tipping point may be lower depending on your organization's tolerance for latency.
Once you determine an ideal active threshold for active connections (example: 100), then we can use that information to determine when scale NGINX Ingress Controller.

### PART 4: Autoscale NGINX Ingress Controller

This part will be focused on the following parts:

- Configure an autoscaling policy using KEDA
- Generate a traffic surge with Locust
- Observe how NGINX Ingress Controller autoscales to cope with the traffic surge

#### Step 1: Install KEDA

Now it's time to build a configuration that autoscales resources as the load (traffic) increases.

For this task, you'll use KEDA, a Kubernetes event-driven autoscaler. KEDA integrates a metrics server (the component that stores and transforms metrics for Kubernetes) and it can consume metrics directly from Prometheus (as well as other tools).

<img width="403" alt="image" src="https://user-images.githubusercontent.com/43017632/158035320-95e4d722-bc5e-4b78-99f6-067ef74d6308.png">

- Add "kedacore" to your repositories with:
```
helm repo add kedacore https://kedacore.github.io/charts
helm install keda kedacore/keda
```
If successful we will get result liket this,
```
NAME: keda
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

#### Step 2: Verify KEDA Installation

```
kubectl get pods
```
<img width="489" alt="image" src="https://user-images.githubusercontent.com/43017632/158035444-dd273880-3e82-440c-b5e1-9a4baa35d59c.png">

To confirm KEDA is running, look for the two KEDA pods (they should be at the top of the list). Note that you should now have the following installed:

KEDA
Locust
NGINX Ingress Controller
Prometheus

#### Step 3: Configuring Autoscaling

We create a yaml for this

Autoscales NGINX Ingress Controller pods from a single pod up to 20 pods.
Uses the 'nginx_connections_active' metric collected by Prometheus to trigger the autoscaling.
Deploys a new pod when the existing pod hits 100 active connections.

```
kubectl apply -f 4-scaled-object.yaml
```

### Step 4: Inspect KEDA HPA
Open the Locust dashboard. If the dashboard is still running from the last challenge, click the Stop button and then click *New test* (in the STATUS column).
This time we'll double the number of active users (from 1000 to 2000) but the other parameters will remain the same:
Number of users: 2000
Spawn rate: 10
Host:

KEDA bridges the metrics collected by Prometheus and feeds them to Kubernetes. It creates a Horizontal Pod Autoscaler (HPA) with those metrics.

Switch back to the terminal tab and manually inspect the KEDA HPA with:

```
kubectl get hpa
```










