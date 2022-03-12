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

### Part 2

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
