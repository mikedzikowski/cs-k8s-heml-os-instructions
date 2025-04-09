
# KAC Example

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
export KACREGISTRY=registry.crowdstrike.com/falcon-kac/us-1/release/falcon-kac


export KAC=$(bash ./falcon-container-sensor-pull.sh -u $FCSCLIENTID -s $FSCSECRET --type falcon-kac --list-tags)
export LATEST_KAC_TAG=$(echo "$KAC" | jq -r '.tags | sort | last')
echo $LATEST_KAC_TAG
export PULLTOKEN=$(bash ./falcon-container-sensor-pull.sh -u $FCSCLIENTID -s $FSCSECRET --type falcon-sensor --get-pull-token)
az account set --subscription $AZURESUBSCRIPTION
az aks get-credentials --resource-group $CLUSTERRESOURCEGROUP --name $CLUSTERNAME --overwrite-existing
helm repo add crowdstrike https://crowdstrike.github.io/falcon-helm
helm repo update
helm install falcon-kac crowdstrike/falcon-kac \
-n falcon-kac --create-namespace \
--set falcon.cid=$FCSCID \
--set image.repository=$KACREGISTRY \
--set image.tag=$LATEST_KAC_TAG \
--set image.registryConfigJSON=$PULLTOKEN
```
