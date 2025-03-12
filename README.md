# Azure External DNS on AKS

This repository provides a guide and resources to set up **External DNS** on Azure Kubernetes Service (AKS) to manage DNS records in **Azure DNS** automatically. External DNS syncs Kubernetes `Ingress` and `Service` resources with Azure DNS, ensuring that your DNS records are always up-to-date with your Kubernetes workloads.

The instructions are based on the blog post: [Kubernetes External DNS for Azure DNS & AKS](https://techcommunity.microsoft.com/blog/coreinfrastructureandsecurityblog/kubernetes-external-dns-for-azure-dns--aks/3809393).

## Table of Contents
- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Setup Instructions](#setup-instructions)
  - [Step 1: Create an Azure DNS Zone](#step-1-create-an-azure-dns-zone)
  - [Step 2: Configure Azure AD App Registration for External DNS](#step-2-configure-azure-ad-app-registration-for-external-dns)
  - [Step 3: Deploy External DNS on AKS](#step-3-deploy-external-dns-on-aks)
  - [Step 4: Test with a Sample Application](#step-4-test-with-a-sample-application)
- [Using External DNS with Kubernetes Services](#using-external-dns-with-kubernetes-services)
- [Usage](#usage)
- [Troubleshooting](#troubleshooting)

## Overview

External DNS is a Kubernetes add-on that automates the management of DNS records by synchronizing exposed Kubernetes `Service` and `Ingress` resources with DNS providers. This guide focuses on integrating External DNS with **Azure DNS** on an AKS cluster, allowing seamless DNS management for your applications.

## Prerequisites

Before you begin, ensure you have the following:

1. **Azure Subscription**: An active Azure subscription.
2. **AKS Cluster**: A running AKS cluster with cluster-admin access.
3. **Azure CLI**: Installed and configured with your Azure credentials.
4. **kubectl**: Installed and configured to communicate with your AKS cluster.
5. **Helm**: Installed for deploying External DNS (optional, if using Helm charts).
6. **Domain Name**: A domain name managed in Azure DNS (e.g., `example.com`, in my case, astute001.com).

## Setup Instructions

### Step 1: Create an Azure DNS Zone

You can create a new Azure DNS Zone with or without delegated domain name. Without delegated domain name means it will not be able to publicly resolve the domain name. But you will still see the created DNS records.

In this lab, I use a delegated domain name: astute001.com. Replace it with your own.

1. Log in to the Azure Portal or use the Azure CLI.
2. Create a DNS zone in Azure DNS for your domain (e.g., `example.com`):
   ```bash
   az dns zone create --resource-group <resource-group-name> --name example.com
   ```
3. Note the DNS zone name and the resource group it belongs to.

4. Update your domain registrar to delegate the domain to Azure DNS name servers. Retrieve the name servers for your DNS zone:
   ```bash
   az dns zone show --resource-group <resource-group-name> --name example.com --query nameServers
   ```

### Step 2: Configure Azure AD App Registration for External DNS

External DNS requires permissions to manage DNS records in Azure DNS. We'll create an Azure AD App Registration and assign it the necessary permissions.

1. **Create an Azure AD App Registration**:
   ```bash
   az ad app create --display-name "ExternalDNS"
   ```
   Note the `appId` from the output.

2. **Create a Service Principal**:
   ```bash
   az ad sp create --id <appId>
   ```
   Note the `objectId` of the service principal.

3. **Assign Permissions to the DNS Zone**:
   Grant the service principal `Contributor` role on the DNS zone:
   ```bash
   az role assignment create --assignee <objectId> --role "Contributor" --scope /subscriptions/<subscription-id>/resourceGroups/<resource-group-name>/providers/Microsoft.Network/dnszones/example.com
   ```

4. **Create a Client Secret**:
   Generate a client secret for the app:
   ```bash
   az ad app credential reset --id <appId> --append
   ```
   Save the `clientSecret` value securely.

5. **Retrieve Tenant ID**:
   Get your Azure tenant ID:
   ```bash
   az account show --query tenantId
   ```

### Step 3: Deploy External DNS on AKS

Now, deploy External DNS on your AKS cluster and configure it to use Azure DNS.

#### Option 1: Using Helm (Recommended)
1. Add the External DNS Helm chart repository:
   ```bash
   helm repo add external-dns https://charts.bitnami.com/bitnami
   helm repo update
   ```

2. Create a `values.yaml` file for Helm with the following content:
   ```yaml
   provider: azure
   azure:
     resourceGroup: <resource-group-name>
     tenantId: <tenant-id>
     subscriptionId: <subscription-id>
     aadClientId: <appId>
     aadClientSecret: <clientSecret>
   domainFilters:
     - example.com
   ```

3. Install External DNS using Helm:
   ```bash
   helm install external-dns external-dns/external-dns -f values.yaml --namespace kube-system
   ```

#### Option 2: Using Kubernetes Manifests
1. Create a JSON file named `azure.json` with the Azure credentials:
   ```json
   {
     "tenantId": "<tenant-id>",
     "subscriptionId": "<subscription-id>",
     "resourceGroup": "<resource-group-name>",
     "aadClientId": "<appId>",
     "aadClientSecret": "<clientSecret>"
   }
   ```

2. Create a Kubernetes Secret from the `azure.json` file:
   ```bash
   kubectl create secret generic azure-config-file --from-file=azure.json --namespace kube-system
   ```

3. Deploy External DNS with a manifest file (example available in the [External DNS GitHub repository](https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/azure.md)):
   ```bash
   kubectl apply -f external-dns.yaml
   ```

Sample Output:

```bash
kubectl get pods,sa -n external-dns
NAME                                READY   STATUS    RESTARTS   AGE
pod/external-dns-767fd784fb-tkrj2   1/1     Running   0          132m

NAME                          SECRETS   AGE
serviceaccount/default        0         134m
serviceaccount/external-dns   0         132m
```
### Step 4: Install nginx ingress controller

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

helm repo update

helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx `
     --create-namespace `
     --namespace ingress-nginx `
     --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-health-probe-request-path"=/healthz
```
Sample Output:

```bash
kubectl get pods,svc -n ingress-nginx
NAME                                            READY   STATUS      RESTARTS   AGE
pod/ingress-nginx-admission-create-vt57z        0/1     Completed   0          87m
pod/ingress-nginx-admission-patch-dfz97         0/1     Completed   1          87m
pod/ingress-nginx-controller-797dfb8dc6-5l8br   1/1     Running     0          87m

NAME                                         TYPE           CLUSTER-IP    EXTERNAL-IP    PORT(S)                      AGE
service/ingress-nginx-controller             LoadBalancer   10.0.247.89   4.157.63.139   80:30328/TCP,443:30452/TCP   87m
service/ingress-nginx-controller-admission   ClusterIP      10.0.176.35   <none>         443/TCP                      87m
```

### Step 5: Test with a Sample Application

1. Deploy a sample application with an `Ingress` resource:
   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: sample-ingress
     annotations:
       kubernetes.io/ingress.class: nginx
   spec:
     rules:
     - host: myapp.example.com
       http:
         paths:
         - path: /
           pathType: Prefix
           backend:
             service:
               name: myapp-service
               port:
                 number: 80
   ```

2. Apply the Ingress resource:
   ```bash
   kubectl apply -f sample-ingress.yaml
   ```

3. Verify that External DNS has created a DNS record in Azure DNS:
   ```bash
   az dns record-set a list --resource-group <resource-group-name> --zone-name example.com
   ```

Sample Output:

```bash
kubectl get pods,svc,ingress
NAME                         READY   STATUS    RESTARTS   AGE
pod/app02-cfd466954-t7fdl    1/1     Running   0          89m

NAME                 TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)        AGE
service/app02-svc    ClusterIP      10.0.149.163   <none>           80/TCP         89m

NAME                                      CLASS   HOSTS                 ADDRESS        PORTS   AGE
ingress.networking.k8s.io/app02-ingress   nginx   app02.astute001.com   4.157.63.139   80      89m
```

4. Access your application using the domain name (e.g., `myapp.example.com`).

In my case, I have created it like: http://app02.astute001.com/

![image](https://github.com/user-attachments/assets/9b068c0c-5fae-49b0-8cf7-b9866fa7d357)


## Using External DNS with Kubernetes Services

In addition to managing DNS records for `Ingress` resources, External DNS can also manage DNS records for Kubernetes `Service` resources, particularly those of type `LoadBalancer`. This is useful when you want to expose a service directly via a public IP and associate it with a DNS name in Azure DNS.

### Configuring External DNS for Services

External DNS can automatically create DNS records for `Service` resources if they have an external IP (e.g., a LoadBalancer service). To enable this, you need to annotate the `Service` with the desired DNS name.

### Example: Expose a Service with External DNS

1. **Create a Sample Service of Type LoadBalancer**:
   Below is an example of a Kubernetes `Service` of type `LoadBalancer` with an annotation for External DNS:
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: myapp-service
     annotations:
       external-dns.alpha.kubernetes.io/hostname: myservice.example.com
   spec:
     type: LoadBalancer
     ports:
     - port: 80
       targetPort: 8080
       protocol: TCP
     selector:
       app: myapp
   ```

2. **Deploy a Sample Deployment**:
   Ensure you have a corresponding `Deployment` to serve traffic:
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: myapp-deployment
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: myapp
     template:
       metadata:
         labels:
           app: myapp
       spec:
         containers:
         - name: myapp
           image: nginx
           ports:
           - containerPort: 8080
   ```

3. **Apply the Resources**:
   Apply the `Deployment` and `Service` to your AKS cluster:
   ```bash
   kubectl apply -f myapp-deployment.yaml
   kubectl apply -f myapp-service.yaml
   ```

4. **Verify the LoadBalancer IP**:
   After the `Service` is created, check its external IP:
   ```bash
   kubectl get svc myapp-service
   ```
   The `EXTERNAL-IP` field will show the public IP assigned by Azure.


Sample Output:

```bash
kubectl get pods,svc
NAME                         READY   STATUS    RESTARTS   AGE
pod/app01-6674ffccc7-njdms   1/1     Running   0          124m

NAME                 TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)        AGE
service/app01-svc    LoadBalancer   10.0.79.122    134.33.140.129   80:32089/TCP   124m

```


5. **Verify the DNS Record in Azure DNS**:
   External DNS will create an `A` record in Azure DNS for `myservice.example.com` pointing to the external IP of the `Service`. Check the DNS record:
   ```bash
   az dns record-set a list --resource-group <resource-group-name> --zone-name example.com
   ```

6. **Access the Service**:
   Once the DNS record propagates, you can access the service using the domain name (e.g., `myservice.example.com`).

In my case, I have created it like: http://app01.astute001.com/

![image](https://github.com/user-attachments/assets/1b218108-53cc-4fff-9643-0a87201cd37b)


### Notes
- Ensure that the `domainFilters` in your External DNS configuration include the domain of the service (e.g., `example.com`).
- You can use additional annotations like `external-dns.alpha.kubernetes.io/ttl` to set a custom TTL for the DNS record.
- If the `Service` is deleted or its external IP changes, External DNS will automatically update or remove the corresponding DNS record.

## Usage

Once External DNS is running, it will automatically monitor your Kubernetes `Ingress` and `Service` resources and update Azure DNS records based on the `host` field and annotations. You can customize its behavior using annotations or configuration options (e.g., `domainFilters`, `txtOwnerId`).

## Troubleshooting

- **DNS Records Not Updating**: Ensure the Azure AD app has the correct permissions and that the `azure.json` file or Helm values are correctly configured.
- **Logs**: Check the External DNS pod logs for errors:
  ```bash
  kubectl logs -l app=external-dns -n kube-system
  ```
- **Domain Filters**: Verify that the `domainFilters` include your domain name.
- **Azure DNS Propagation**: DNS changes may take a few minutes to propagate.

Let us check the Azure DNS zone configuration. Note the A records was added with public IP for service and ingress controller.

![image](https://github.com/user-attachments/assets/716b34ca-41a3-4665-8f20-b8580727b407)
