```
---
az login --use-device-code 
az extension add --name aks-preview
az extension update --name aks-preview
az feature register --namespace "Microsoft.ContainerService" --name "NodeAutoProvisioningPreview"
az feature show --namespace "Microsoft.ContainerService" --name "NodeAutoProvisioningPreview"
az provider register --namespace Microsoft.ContainerService
---
az group create --name aks-nap-rg --location eastus
az aks create --name ga-nap-aks --resource-group aks-nap-rg --node-provisioning-mode Auto --network-plugin azure --network-plugin-mode overlay --network-dataplane cilium --nodepool-taints CriticalAddonsOnly=true:NoSchedule
---
```

```
kubectl get nodes -o json | jq '.items[] | {name: .metadata.name, instance_type: .metadata.labels["node.kubernetes.io/instance-type"], nodepool_type: .metadata.labels["kubernetes.azure.com/nodepool-type"], karpeneter_nodepool: .metadata.labels ["karpenter.sh/nodedpool"], 
topology_spread: .metadata.labels["topology.kubernetes.io/zone"], karpeneter_nodepool: .metadata.labels ["karpenter.sh/capacity-type"],image_version: .metadata.labels["kubernetes.azure.com/node-image-version"],  }'
```

```
alias k=kubectl
k get no -o wide
kubectl get events -A --field-selector source=karpenter -w
kubectl get NodePool default -o yaml
kubectl edit aksnodeclass default
```

```
# to scale the pods so that we can see the karpenter events kicks in to add the nodes 
kubectl scale deploy -n global-azure-ns --replicas=20 --all
```
