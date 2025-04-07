# cs-k8s-heml-os-instructions
A simple markdown repo with instructions and resources on how to deploy the CrowdStrike security stack on K8s and OpenShift clusters 


# CrowdStrike Image Assessment Runtime (IAR) Deployment Guide

This guide provides instructions for deploying CrowdStrike IAR on both OpenShift and standard Kubernetes environments.

## Prerequisites for All Deployments

- CrowdStrike Falcon account with appropriate permissions
- CrowdStrike API credentials (Client ID and Secret)
- Your CrowdStrike Customer ID (CID)

## Creating CrowdStrike API Credentials

1. **Log in to the CrowdStrike Falcon Console** at https://falcon.crowdstrike.com/

2. **Access API Clients & Keys**
   - Go to **Support & Resources** menu
   - Select **API Clients & Keys**

3. **Create New API Client**
   - Click **Create API Client** button
   - Enter a descriptive name (e.g., "Falcon Image Analyzer")
   - Select the required permissions:
     - **Falcon Container CLI**: `Write`
     - **Falcon Container Image**: `Read/Write`
     - **Falcon Images Download**: `Read`

4. **Save Credentials**
   - Click **Create**
   - **IMPORTANT**: Copy and securely store both the Client ID and Client Secret
   - These credentials will only be shown once

---

## Deployment Option 1: OpenShift Web Console

### Prerequisites
- OpenShift 4.x+ cluster with administrator access

### Installation Steps

#### 1. Install Falcon Operator
1. Go to **Operators** → **OperatorHub**
2. Search for "CrowdStrike"
3. Click on **CrowdStrike Falcon Operator** → **Install**
4. Keep all defaults → Click **Install**
5. Wait for status to show "Succeeded"

#### 2. Deploy Image Analyzer
1. Go to **Operators** → **Installed Operators** → **CrowdStrike Falcon Operator**
2. Click on **Falcon Image Analyzer** tab → **Create Instance**
3. Use Form View:
   - **Name**: `falcon-image-analyzer`
   - **Namespace**: `falcon-image-analyzer`
   - **CID**: Your CrowdStrike Customer ID
   - **Cloud Region**: `autodiscover`
   - Under **API Credentials**:
     - For **Client ID**, enter your Falcon OAuth2 API Client ID that you created earlier
     - For **Client Secret**, enter your Falcon OAuth2 API Client Secret that you created earlier

4. Click **Create**

#### 3. Verify Deployment
1. Go to **Workloads** → **Pods**
2. Filter by namespace: `falcon-image-analyzer`
3. Confirm Image Analyzer pods show "Running" status

---

## Deployment Option 2: OpenShift CLI

### Prerequisites
- OpenShift 4.x+ cluster
- `oc` CLI tool installed and configured
- Cluster administrator privileges

### Installation Steps

#### 1. Install Falcon Operator
```bash
# Create the operator
oc apply -f https://github.com/CrowdStrike/falcon-operator/releases/latest/download/falcon-operator.yaml

# Verify the operator is running
oc get pods -n falcon-operator-system
```

#### 2. Create Namespace and API Credentials Secret
```bash
# Create namespace
oc create namespace falcon-image-analyzer

# Create API credentials secret
oc create secret generic falcon-api-key \
  --namespace falcon-image-analyzer \
  --from-literal=client_id=YOUR_CLIENT_ID \
  --from-literal=client_secret=YOUR_CLIENT_SECRET
```

#### 3. Deploy Image Analyzer
Create a file named `falcon-image-analyzer.yaml`:

```yaml
apiVersion: falcon.crowdstrike.com/v1alpha1
kind: FalconImageAnalyzer
metadata:
  name: falcon-image-analyzer
  namespace: falcon-image-analyzer
spec:
  # If you want the operator to automatically determine your CID
  # using the provided API credentials, you can omit the CID field
  # and just specify the cloud region
  falcon:
    cloud: autodiscover  # or us-1, us-2, eu-1, etc.
    credentials:
      api:
        clientId:
          secretKeyRef:
            name: falcon-api-key
            key: client_id
        clientSecret:
          secretKeyRef:
            name: falcon-api-key
            key: client_secret
```

If you prefer to specify your CID explicitly:

```yaml
apiVersion: falcon.crowdstrike.com/v1alpha1
kind: FalconImageAnalyzer
metadata:
  name: falcon-image-analyzer
  namespace: falcon-image-analyzer
spec:
  falcon:
    cid: YOUR_CROWDSTRIKE_CID
    cloud: autodiscover
    credentials:
      api:
        clientId:
          secretKeyRef:
            name: falcon-api-key
            key: client_id
        clientSecret:
          secretKeyRef:
            name: falcon-api-key
            key: client_secret
```

Apply the configuration:
```bash
oc apply -f falcon-image-analyzer.yaml
```

#### 4. Verify Deployment
```bash
# Check the FalconImageAnalyzer resource status
oc get falconimageanalyzer -n falcon-image-analyzer

# Check that pods are running
oc get pods -n falcon-image-analyzer
```

#### 5. Monitor Deployment Progress
```bash
# Watch the operator logs
oc logs -f deployment/falcon-operator-controller-manager -n falcon-operator-system

# Check the status of the FalconImageAnalyzer resource
oc describe falconimageanalyzer falcon-image-analyzer -n falcon-image-analyzer
```

---

## Deployment Option 3: Standard Kubernetes and Rancher with Helm

### Prerequisites
- Kubernetes cluster (any distribution) or Rancher-managed cluster
- Helm 3.x installed
- `kubectl` configured to access your cluster

### Installation Steps

#### 1. Add CrowdStrike Helm Repository
```bash
helm repo add crowdstrike https://crowdstrike.github.io/falcon-helm
helm repo update
```

#### 2. Create Namespace
```bash
kubectl create namespace falcon-image-analyzer
```

#### 3. Create API Credentials Secret
```bash
kubectl create secret generic falcon-api-key \
  --namespace falcon-image-analyzer \
  --from-literal=client_id=YOUR_CLIENT_ID \
  --from-literal=client_secret=YOUR_CLIENT_SECRET
```

#### 4. Create Configuration File
Create a file named `config_values.yaml` with your CrowdStrike configuration:

```yaml
falcon:
  cid: YOUR_CROWDSTRIKE_CID
  cloud: autodiscover
  credentials:
    api:
      clientIdSecretRef:
        name: falcon-api-key
        key: client_id
      clientSecretSecretRef:
        name: falcon-api-key
        key: client_secret

scanSettings:
  scanOnPush: true
  scanOnPull: true
```

#### 5. Install Image Analyzer
```bash
helm upgrade --install -f config_values.yaml \
  --namespace falcon-image-analyzer imageanalyzer crowdstrike/falcon-image-analyzer
```

#### 6. Verify Deployment
```bash
kubectl get pods -n falcon-image-analyzer
```

You should see pods with names starting with `imageanalyzer-falcon-image-analyzer-` in Running state.

---

## Available CrowdStrike Helm Charts

CrowdStrike offers several Helm charts for different components of the Falcon platform. All charts are available in the [CrowdStrike Falcon Helm repository](https://github.com/CrowdStrike/falcon-helm).

### 1. [falcon-image-analyzer](https://github.com/CrowdStrike/falcon-helm/tree/main/helm-charts/falcon-image-analyzer)
- **Purpose**: Deploys the Image Assessment Runtime (IAR) for container image scanning
- **Key Features**: 
  - Scans container images for vulnerabilities and malware
  - Supports scanning on push, pull, or periodic schedules
  - Integrates with container registries

### 2. [falcon-sensor](https://github.com/CrowdStrike/falcon-helm/tree/main/helm-charts/falcon-sensor)
- **Purpose**: Deploys the CrowdStrike Falcon sensor for Kubernetes node protection
- **Key Features**:
  - Provides runtime protection for Kubernetes nodes
  - Detects and prevents threats at the host level
  - Supports various Linux distributions

### 3. [falcon-container](https://github.com/CrowdStrike/falcon-helm/tree/main/helm-charts/falcon-container)
- **Purpose**: Deploys the CrowdStrike Container Security components
- **Key Features**:
  - Provides runtime protection for containers
  - Detects and prevents container-based threats
  - Supports Kubernetes admission controller for pre-runtime checks
- **Note**: This chart should be used for sidecar deployment scenarios where container-level protection is required rather than node-level protection

### 4. [falcon-admission-controller](https://github.com/CrowdStrike/falcon-helm/tree/main/helm-charts/falcon-admission-controller)
- **Purpose**: Deploys the Kubernetes admission controller for container security
- **Key Features**:
  - Enforces security policies before containers are deployed
  - Blocks deployment of vulnerable or non-compliant images
  - Integrates with Image Assessment Runtime for scanning results

### 5. [falcon-kac](https://github.com/CrowdStrike/falcon-helm/tree/main/helm-charts/falcon-kac)
- **Purpose**: Deploys the Kubernetes Admission Controller (KAC)
- **Key Features**:
  - Newer version of the admission controller with enhanced features
  - Supports more granular policy enforcement
  - Improved integration with CrowdStrike cloud

---

## Troubleshooting

### Common Issues
1. **Pods not starting:**
   - Verify API credentials are correct
   - Ensure CID is in proper format
   - Check pod logs for detailed errors

2. **Authentication failures:**
   - Confirm API client has required permissions
   - Verify secret values are correctly referenced

3. **Scanning not working:**
   - Check connectivity to CrowdStrike cloud
   - Verify scan settings are properly configured

### Viewing Logs
```bash
kubectl logs -n falcon-image-analyzer <pod-name>
```

## Uninstalling

**OpenShift Operator:**
```bash
oc delete falconimageanalyzer falcon-image-analyzer -n falcon-image-analyzer
oc delete -f https://github.com/CrowdStrike/falcon-operator/releases/latest/download/falcon-operator.yaml
```

**Helm (Standard Kubernetes):**
```bash
helm uninstall -n falcon-image-analyzer imageanalyzer
```

## Additional Resources
- [Falcon Operator Guide for Image Analyzer](https://github.com/CrowdStrike/falcon-operator/tree/main/docs/resources/imageanalyzer) - Detailed documentation for OpenShift deployments
- [Falcon Helm Chart Documentation](https://github.com/CrowdStrike/falcon-helm/tree/main/helm-charts/falcon-image-analyzer) - Complete reference for Kubernetes Helm deployments
- [CrowdStrike Falcon Helm Repository](https://github.com/CrowdStrike/falcon-helm) - Source for all CrowdStrike Helm charts
