# Create and Manage Cloud Resources: Challenge Lab (GSP313)

![Google Cloud](https://img.shields.io/badge/Google_Cloud-4285F4?logo=googlecloud&logoColor=white)
![Lab](https://img.shields.io/badge/Lab-GSP313-34A853)
![Level](https://img.shields.io/badge/Level-Introductory-blue)

Reference solution for the Google Cloud Skills Boost challenge lab **Create and Manage Cloud Resources** (GSP313).

## 📖 Overview

The lab checks that you can provision core Google Cloud infrastructure from the command line:

1. A Compute Engine jumphost VM.
2. A Google Kubernetes Engine (GKE) cluster running a sample app.
3. An HTTP load balancer in front of a managed instance group of NGINX web servers.

## 🧰 Services used

Compute Engine · Google Kubernetes Engine · Instance templates & managed instance groups · Cloud Load Balancing · VPC firewall rules

## ✅ Prerequisites

- Open **Cloud Shell** in the lab project.
- Copy the instance name, app port and firewall rule name from your lab instructions into the variables below.

```bash
export INSTANCE_NAME="<jumphost-instance-name>"
export APP_PORT="<app-port>"
export FIREWALL_RULE="<firewall-rule-name>"
export REGION="us-east1"
export ZONE="us-east1-b"
export MACHINE_TYPE="e2-micro"   # use the machine type your lab specifies

gcloud config set compute/region $REGION
gcloud config set compute/zone   $ZONE
```

## 🚀 Solution

### Task 1: Create the jumphost instance

```bash
gcloud compute instances create $INSTANCE_NAME \
  --network nucleus-vpc \
  --zone $ZONE \
  --machine-type $MACHINE_TYPE \
  --image-family debian-12 \
  --image-project debian-cloud
```

### Task 2: Create a GKE cluster and deploy the app

```bash
gcloud container clusters create nucleus-cluster --zone $ZONE
gcloud container clusters get-credentials nucleus-cluster --zone $ZONE

kubectl create deployment hello-server --image=gcr.io/google-samples/hello-app:2.0
kubectl expose deployment hello-server --type=LoadBalancer --port $APP_PORT
```

### Task 3: Set up an HTTP load balancer

**Startup script** that installs NGINX and shows the hostname:

```bash
cat << 'EOS' > startup.sh
#!/bin/bash
apt-get update
apt-get install -y nginx
service nginx start
sed -i -- 's/nginx/Google Cloud Platform - '"$HOSTNAME"'/' /var/www/html/index.nginx-debian.html
EOS
```

**Instance template and managed instance group:**

```bash
gcloud compute instance-templates create web-server-template \
  --metadata-from-file startup-script=startup.sh \
  --network nucleus-vpc \
  --machine-type $MACHINE_TYPE \
  --region $REGION

gcloud compute instance-groups managed create web-server-group \
  --base-instance-name web-server \
  --size 2 \
  --template web-server-template \
  --region $REGION
```

**Firewall rule and health check:**

```bash
gcloud compute firewall-rules create $FIREWALL_RULE \
  --network nucleus-vpc \
  --allow tcp:80

gcloud compute http-health-checks create http-basic-check

gcloud compute instance-groups managed set-named-ports web-server-group \
  --named-ports http:80 \
  --region $REGION
```

**Backend service, URL map, proxy and forwarding rule:**

```bash
gcloud compute backend-services create web-server-backend \
  --protocol HTTP \
  --http-health-checks http-basic-check \
  --global

gcloud compute backend-services add-backend web-server-backend \
  --instance-group web-server-group \
  --instance-group-region $REGION \
  --global

gcloud compute url-maps create web-server-map --default-service web-server-backend

gcloud compute target-http-proxies create http-lb-proxy --url-map web-server-map

gcloud compute forwarding-rules create http-content-rule \
  --global \
  --target-http-proxy http-lb-proxy \
  --ports 80

gcloud compute forwarding-rules list
```

> The load balancer can take 5 to 10 minutes to start serving traffic.

## 🏗️ Architecture

```text
Client ──► Forwarding rule :80 ──► HTTP proxy ──► URL map ──► Backend service
                                                                   │
                                                  Managed instance group (2× NGINX VMs)
```

---

## ⚠️ Notes

- Resource names, zones, regions and ports are generated per lab session. Always use the values shown in **your** lab instructions.
- These steps were written when the lab was live. Google revises labs over time, so the current version may differ.

## 📌 Disclaimer

This repository is a study reference. Attempt the lab on your own first, then compare approaches. It is not affiliated with or endorsed by Google.

## 👤 Author

**Girish Sharma** · Cloud & DevOps Engineer · [GitHub](https://github.com/codewithgarry) · [LinkedIn](https://www.linkedin.com/in/codewithgarry/)
