# AutoscalingKubernetes-HandleHeavyTraffic

## Problem Scenario


## Description and Technology Used


## Solution


### Part 1


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

#### Step 2: Ingress manifest to route traffic to your app

