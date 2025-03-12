Below is the updated GitHub README with a new section titled "Using External DNS with Kubernetes Services." This section explains how to configure External DNS to work with Kubernetes `Service` resources (e.g., LoadBalancer services) to manage DNS records in Azure DNS. The rest of the README remains unchanged but is included for completeness.

---

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
- [Contributing](#contributing)
- [License](#license)

## Overview

External DNS is a Kubernetes add-on that automates the management of DNS records by synchronizing exposed Kubernetes `Service` and `Ingress` resources with DNS providers. This guide focuses on integrating External DNS with **Azure DNS** on an AKS cluster, allowing seamless DNS management for your applications.

## Prerequisites

Before you begin, ensure you have the following:

1. **Azure Subscription**: An active Azure subscription.
2. **AKS Cluster**: A running AKS cluster with cluster-admin access.
3. **Azure CLI**: Installed and configured with your Azure credentials.
4. **kubectl**: Installed and configured to communicate with your AKS cluster.
5. **Helm**: Installed for deploying External DNS (optional, if using Helm charts).
6. **Domain Name**: A domain name managed in Azure DNS (e.g., `example.com`).

## Setup Instructions

### Step 1: Create an Azure DNS Zone

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

### Step 4: Test with a Sample Application

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

4. Access your application using the domain name (e.g., `myapp.example.com`).

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

5. **Verify the DNS Record in Azure DNS**:
   External DNS will create an `A` record in Azure DNS for `myservice.example.com` pointing to the external IP of the `Service`. Check the DNS record:
   ```bash
   az dns record-set a list --resource-group <resource-group-name> --zone-name example.com
   ```

6. **Access the Service**:
   Once the DNS record propagates, you can access the service using the domain name (e.g., `myservice.example.com`).

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

## Contributing

Contributions are welcome! Feel free to submit a pull request or open an issue for bugs, feature requests, or improvements.

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.

---

### Changes Made
- Added a new section titled "Using External DNS with Kubernetes Services" that explains how to configure External DNS to manage DNS records for Kubernetes `Service` resources of type `LoadBalancer`.
- Included a step-by-step example with YAML manifests for a `Service` and `Deployment`, along with instructions to verify and access the service.
- Added notes about annotations and behavior of External DNS with services.

Let me know if you'd like further refinements or additional details!
