
# Sensor Example

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
export SENSORREGISTRY=registry.crowdstrike.com/falcon-sensor/us-1/release/falcon-sensor


echo $SENSORREGISTRY
echo $FCSCLIENTID
echo $FSCSECRET
curl -sSL -o falcon-container-sensor-pull.sh https://github.com/CrowdStrike/falcon-scripts/releases/latest/download/falcon-container-sensor-pull.sh
chmod 777 ./falcon-container-sensor-pull.sh
export SENSOR=$(bash ./falcon-container-sensor-pull.sh -u $FCSCLIENTID -s $FSCSECRET --type falcon-sensor --list-tags)
export LATEST_SENSOR_TAG=$(echo "$SENSOR" | jq -r '.tags | sort | last')
export PULLTOKEN=$(bash ./falcon-container-sensor-pull.sh -u $FCSCLIENTID -s $FSCSECRET --type falcon-sensor --get-pull-token)
echo $PULLTOKEN
az account set --subscription $AZURESUBSCRIPTION
az aks get-credentials --resource-group $CLUSTERRESOURCEGROUP --name $CLUSTERNAME --overwrite-existing
helm repo add crowdstrike https://crowdstrike.github.io/falcon-helm
helm repo update
helm upgrade --install falcon-helm crowdstrike/falcon-sensor \
-n falcon-system --create-namespace \
--set falcon.cid=$FCSCID \
--set node.image.repository=$SENSORREGISTRY \
--set node.image.tag=$LATEST_SENSOR_TAG \
--set node.image.registryConfigJSON=$PULLTOKEN
```
