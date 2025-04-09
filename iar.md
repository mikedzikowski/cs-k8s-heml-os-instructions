
# CrowdStrike Image Assessment Runtime (IAR) Deployment Guide

This guide provides instructions for deploying CrowdStrike IAR on both OpenShift and standard Kubernetes environments.

## Prerequisites for All Deployments

- CrowdStrike Falcon account with appropriate permissions
- CrowdStrike API credentials (Client ID and Secret)
- Your CrowdStrike Customer ID (CID)

## Creating CrowdStrike API Credentials

1. **Log in to the CrowdStrike Falcon Console** at <https://falcon.crowdstrike.com/>

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
   - Under **Falcon Platform API Configuration**:
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

#### 7. Example

```bash
# pull script
curl -sSL -o falcon-container-sensor-pull.sh https://github.com/CrowdStrike/falcon-scripts/releases/latest/download/falcon-container-sensor-pull.sh

# Update permissions
chmod 777 ./falcon-container-sensor-pull.sh

export FCSCLIENTID=abcdef1234567890abcdef1234567890
export FSCSECRET=abcdef1234567890abcdef1234567890abcdef12
export CLUSTERRESOURCEGROUP=my-resource-group
export CLUSTERNAME=my-aks-cluster
export FCSCID=ABCDEF1234567890ABCDEF1234567890-12
export AZURESUBSCRIPTION=00000000-0000-0000-0000-000000000000
export IARREGISTRY=registry.crowdstrike.com/falcon-imageanalyzer/us-1/release/falcon-imageanalyzer

# Deploy IAR
export IAR=$(bash ./falcon-container-sensor-pull.sh -u $FCSCLIENTID -s $FSCSECRET --type falcon-imageanalyzer --list-tags)
export LATEST_IAR_TAG=$(echo "$IAR" | jq -r '.tags | sort | last')
export PULLTOKEN=$(bash ./falcon-container-sensor-pull.sh -u $FCSCLIENTID -s $FSCSECRET --type falcon-imageanalyzer --get-pull-token)
#az account set --subscription $AZURESUBSCRIPTION
#az aks get-credentials --resource-group $CLUSTERRESOURCEGROUP --name $CLUSTERNAME --overwrite-existing
helm repo add crowdstrike https://crowdstrike.github.io/falcon-helm
helm repo update
helm upgrade --install iar crowdstrike/falcon-image-analyzer -n falcon-image-analyzer --create-namespace \
--set deployment.enabled=true \
--set crowdstrikeConfig.clusterName=$CLUSTERNAME \
--set crowdstrikeConfig.clientID=$FCSCLIENTID \
--set crowdstrikeConfig.clientSecret=$FSCSECRET \
--set crowdstrikeConfig.agentRegion=us-1 \
--set image.registryConfigJSON=$PULLTOKEN \
--set image.repository=$IARREGISTRY --set image.tag=$LATEST_IAR_TAG \
--set crowdstrikeConfig.cid=$FCSCID

"kubectl -n falcon-image-analyzer get pods"
michael [ ~ ]$ kubectl -n falcon-image-analyzer get pods
NAME                                        READY   STATUS    RESTARTS   AGE
iar-falcon-image-analyzer-d8b8596f8-hr99c   1/1     Running   0          9s
```
## View Logs 


```bash
kubectl logs IAR-POD-NAME -n falcon-image-analyzer

Example:
time="2025-04-09T18:17:46Z" level=info msg="received scan request event" image_name="docker.io/devopsfaith/krakend:latest" mode=watcher pod_namespace=default pod_name=krakend-66f8f7c6d9-sq5vw image_digest=6ab066d99a8df9991b22f025829a23b69d41b9d012103235a3491a45b223f68c
time="2025-04-09T18:17:46Z" level=info msg="image not found in cloud. scan needed" image_digest=6ab066d99a8df9991b22f025829a23b69d41b9d012103235a3491a45b223f68c mode=watcher pod_namespace=default pod_name=krakend-66f8f7c6d9-sq5vw image_name="docker.io/devopsfaith/krakend:latest"
time="2025-04-09T18:17:46Z" level=info msg="scanning new image" mode=watcher pod_namespace=default pod_name=krakend-66f8f7c6d9-sq5vw image_digest=6ab066d99a8df9991b22f025829a23b69d41b9d012103235a3491a45b223f68c image_name="docker.io/devopsfaith/krakend:latest"
time="2025-04-09T18:17:46Z" level=info msg="processing image" pod_namespace=default pod_name=krakend-66f8f7c6d9-sq5vw image_name="docker.io/devopsfaith/krakend:latest" image_digest=6ab066d99a8df9991b22f025829a23b69d41b9d012103235a3491a45b223f68c mode=watcher
time="2025-04-09T18:17:46Z" level=info msg="No registry credentials found from cloud provider" mode=watcher pod_namespace=default pod_name=krakend-66f8f7c6d9-sq5vw image_name="docker.io/devopsfaith/krakend:latest" image_digest=6ab066d99a8df9991b22f025829a23b69d41b9d012103235a3491a45b223f68c
time="2025-04-09T18:17:46Z" level=info msg="extracting image" pod_namespace=default pod_name=krakend-66f8f7c6d9-sq5vw image_name="docker.io/devopsfaith/krakend:latest" image_digest=6ab066d99a8df9991b22f025829a23b69d41b9d012103235a3491a45b223f68c mode=watcher
time="2025-04-09T18:17:48Z" level=info msg="copied remote image : {\"schemaVersion\":2,\"mediaType\":\"application/vnd.docker.distribution.manifest.v2+json\",\"config\":{\"mediaType\":\"application/vnd.docker.container.image.v1+json\",\"size\":2479,\"digest\":\"sha256:7937b5041df314398f0a6b5904efd0180ffa6feccdefcf97a1933b462301aea5\"},\"layers\":[{\"mediaType\":\"application/vnd.docker.image.rootfs.diff.tar\",\"size\":8120832,\"digest\":\"sha256:08000c18d16dadf9553d747a58cf44023423a9ab010aab96cf263d2216b8b350\"},{\"mediaType\":\"application/vnd.docker.image.rootfs.diff.tar\",\"size\":2048,\"digest\":\"sha256:9c359671bddf1265f09062d49140ead3b97d7a453469fef9c507d11651d7baf6\"},{\"mediaType\":\"application/vnd.docker.image.rootfs.diff.tar\",\"size\":1710080,\"digest\":\"sha256:1a197abc4fd5ae1eec74c29de27c1b2617038e56b29b300a9821e5b01bb7348d\"},{\"mediaType\":\"application/vnd.docker.image.rootfs.diff.tar\",\"size\":116560896,\"digest\":\"sha256:ebddfa17398c8bed6335677496448de19bff6fe73d5c0decc22f3a83b2d792d1\"},{\"mediaType\":\"application/vnd.docker.image.rootfs.diff.tar\",\"size\":3072,\"digest\":\"sha256:2a2bf1923ead609a477a7d7a366f1558cfe5056ef023e5973fa43f1e53050038\"}]}" pod_namespace=default pod_name=krakend-66f8f7c6d9-sq5vw image_name="docker.io/devopsfaith/krakend:latest" image_digest=6ab066d99a8df9991b22f025829a23b69d41b9d012103235a3491a45b223f68c mode=watcher

```

kubectl logs IAR-POD-NAME -n falcon-image-analyzer
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
