# Kubernetes cluster for JupyterHub

Set the following environment variables as needed
```bash
export CLUSTER_NAME=""
export RESOURCE_GROUP=""
export NODE_RESOURCE_GROUP=""
export LOCATION=""
export STORAGE_ACCOUNT_NAME=""
export STORAGE_KEY=""
```

Choose the applicable node size from here [VM Sizes](https://docs.microsoft.com/en-us/azure/cloud-services/cloud-services-sizes-specs#dv2-series)

### Price List (japaneast)
| Instance | vCPUs | Memory | Monthly Price |
|-|-|-|-|
|Standard_B2s|2|4|39.71$|
|Standard_A2m_v2|2|16|111.69$|
|Standard_D4_v3|4|16|188.34|
|Standard_D4s_v3|4|16|188.34|

---
## Cluster Creation and Credentials

First of all create a directory to wrap everything.

Create an SSH key for the cluster
```ssh-keygen -f ssh-key-hub```

Create the cluster

This will create one node as the master node that's managed by Azure directly.

```bash
az aks create --name $CLUSTER_NAME \
              --resource-group $RESOURCE_GROUP \
              --ssh-key-value .ssh-key-hub.pub \
              --node-count 1 \
              --node-vm-size Standard_B2s \
              --node-osdisk-size 30 \
              --node-resource-group $NODE_RESOURCE_GROUP \
              --location $LOCATION \
              --output table
```

Install kubectl if not installed already
```az aks install-cli```

Get credentials for kubectl
```bash
az aks get-credentials --name $CLUSTER_NAME \
                       --resource-group $RESOURCE_GROUP \
                       --output table
```

---
## Preparing the shared data

### Installing Blobfuse FlexVolume
Install using kubernetes daemonset
```bash
kubectl apply -f https://raw.githubusercontent.com/Azure/kubernetes-volume-drivers/master/flexvolume/blobfuse/deployment/blobfuse-flexvol-installer-1.9.yaml
```

#### Check daemonset status
```bash
watch kubectl describe daemonset blobfuse-flexvol-installer \
      --namespace=kube-system
```

#### Create secret for account name and key
```bash
kubectl create secret generic blobfusecreds \
        --from-literal accountname=$STORAGE_ACCOUNT_NAME \
        --from-literal accountkey=$STORAGE_KEY \
        --type="azure/blobfuse"
```

#### Create a PV (Persistant Volume)

Use the template [pv-blobfuse-flexvol.yaml](Templates/pv-blobfuse-flexvol.yaml)
and modify it to your needs. Basic configuration would look something like this:
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-blobfuse-flexvol
spec:
  capacity:
    storage: 20Gi
  accessModes:
    - ReadOnlyMany
  persistentVolumeReclaimPolicy: Retain
  flexVolume:
    driver: "azure/blobfuse"
    secretRef:
      name: blobfusecreds
    options:
      container: # Specify the container name here
      tmppath: /tmp/blobfuse
      mountoptions: "--file-cache-timeout-in-seconds=120"
```

Create the pod
```bash
kubectl create -f pv-blobfuse-flexvol.yaml
```

#### Create a PVC (Persistant Volume Claim)

Same as PV, use the template [pvc-blobfuse-flexvol.yaml](Templates/pvc-blobfuse-flexvol.yaml)
and modify to match what's needed with the previous one:

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvc-blobfuse-flexvol
spec:
  accessModes:
    - ReadOnlyMany
  resources:
    requests:
      storage: 20Gi
  volumeName: pv-blobfuse-flexvol
  storageClassName: ""
```

Then create the pod:
```bash
kubectl create -f pvc-blobfuse-flexvol.yaml
```
Check status for both PV and PVC until it changes from ```Pending``` to ```Bound```

```bash
kubectl get pv
kubectl get pvc
```

#### Create the pod with blobfuse flexvolume PVC

Finally, use [nginx-flex-blobfuse-pvc.yaml](Templates/nginx-flex-blobfuse-pvc.yaml)
and edit to match what's needed with the PV and PVC:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-flex-blobfuse
spec:
  containers:
  - name: nginx-flex-blobfuse
    image: nginx
    volumeMounts:
    - name: flexvol-mount
      mountPath: /data
  volumes:
  - name: flexvol-mount
    persistentVolumeClaim:
      claimName: pvc-blobfuse-flexvol
      readOnly: true
```

Create pod
```bash
kubectl create -f nginx-flex-blobfuse-pvc.yaml
```

Check status with
```bash
kubectl describe po nginx-flex-blobfuse
```

Enter the pod container and check if the folder mounted correctly on ```'/'```
```bash
kubectl exec -it nginx-flex-blobfuse -- bash
```
---
## Kubernetes Dashboard

To access Kubernetes dashboard using ```az aks browse```
```bash
kubectl create clusterrolebinding kubernetes-dashboard --clusterrole=cluster-admin --serviceaccount=kube-system:kuber
netes-dashboard
```

And that's it for preliminary setup, now on to JupyterHub.

---
## Installing Helm and JupyterHub

```bash
# MacOS
brew install kubernetes-helm

# Windows
choco install kubernetes-helm
```

Setup a service account
```bash
kubectl --namespace kube-system create serviceaccount tiller
```

Give serviceaccount full permissions to manage the cluster
```bash
kubectl create clusterrolebinding tiller \
        --clusterrole cluster-admin \
        --serviceaccount=kube-system:tiller
```

Initialize Helm and Tiller
```bash
helm init --service-account tiller --wait
```

Secure Helm
```bash
kubectl patch deployment tiller-deploy --namespace=kube-system --type=json --patch='[{"op": "add", "path": "/spec/template/spec/containers/0/command", "value": ["/tiller", "--listen=localhost:44134"]}]'
```

### JupyterHub Helm Configuration File
Generate a random key
``` openssl rand -hex 32 ```

Create [jupyterhub.yaml](templates/jupyterhub.yaml) and make sure to include the
following
```yaml
singleuser:
  storage:
    type: none
    extraVolumes:
    - name: jupyterhub-shared
      persistentVolumeClaim:
        claimName: pvc-blobfuse-flexvol
    extraVolumeMounts:
    - name: jupyterhub-shared
      mountPath: /home/jovyan/data
      readOnly: true # Forgetting to set this will give users full access to the blob
```

Add JupyterHub repo to Helm:
```bash
helm repo add jupyterhub https://jupyterhub.github.io/helm-chart/
helm repo update
```

And finally run install and run JupyterHub with
```bash
helm upgrade --install jhub jupyterhub/jupyterhub \
             --version=0.8.2 \
             --values jupyterhub.yaml
```

Watch the pods 
```bash
kubectl get pod
```

Watch the services and wait until the external IP is published
```bash
kubectl get service
```

## Setting A Records in Azure DNS

After deploying JupyterHub get the external ip address
```bash
az network public-ip list -o tsv -g $NODE_RESOURCE_GROUP --query "[].id"
```

Then update the ```hub``` A record with it.
```bash
az network dns record-set a update \
                            -g DNS \
                            -n hub \
                            -z mohi.ai \
                            --target-resource $(az network public-ip list -o tsv -g $NODE_RESOURCE_GROUP --query "[].id")
```

## Cleanup
```bash
helm delete jhub --purge
kubectl delete --all pods
kubectl delete --all pvc
kubectl delete --all pv
az group delete --name $NODE_RESOURCE_GROUP
````