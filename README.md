STEPS FOR ENABLING AKS NODE AUTO-PROVISIONING
==============================================

```
  az login --use-device-code 
  az extension add --name aks-preview
  az extension update --name aks-preview
  az feature register --namespace "Microsoft.ContainerService" --name "NodeAutoProvisioningPreview"
  az feature show --namespace "Microsoft.ContainerService" --name "NodeAutoProvisioningPreview"
  az provider register --namespace Microsoft.ContainerService
  alias k=kubectl 
---
  az group create --name aks-nap-rg --location eastus
  az aks create --name ga-nap-aks --resource-group aks-nap-rg --node-provisioning-mode Auto --network-plugin azure --network-plugin-mode overlay --network-dataplane cilium --nodepool-taints CriticalAddonsOnly=true:NoSchedule

```

---

```
k get nodes -o json | jq '.items[] | {name: .metadata.name, instance_type: .metadata.labels["node.kubernetes.io/instance-type"], nodepool_type: .metadata.labels["kubernetes.azure.com/nodepool-type"], karpeneter_nodepool: .metadata.labels ["karpenter.sh/nodedpool"], 
topology_spread: .metadata.labels["topology.kubernetes.io/zone"], karpeneter_nodepool: .metadata.labels ["karpenter.sh/capacity-type"],image_version: .metadata.labels["kubernetes.azure.com/node-image-version"],  }'
```

```
k get no -o wide

k get NodePool default -o yaml
k edit aksnodeclass default
```

```
# scale/generate load test to see the karpenter events
k scale deploy -n global-azure-ns --replicas=20 --all

# keep this in separate window to see it in action
k get events -A --field-selector source=karpenter -w

#additional commands
kubectl config get-contexts
```

```
Cluster Auto scaler
===================
k config get-contexts
k get cm -n kube-system cluster-autoscaler-status -o yaml
k get events -A --field-selector source=cluster-autoscaler -w

k create namespace inflatens
k apply -f inflate.yaml
k get all -n inflatens

k get nodes -o json | jq '.items[] | {name: .metadata.name, instance_type: .metadata.labels["node.kubernetes.io/instance-type"], nodepool_type: .metadata.labels["kubernetes.azure.com/nodepool-type"], topology_spread: .metadata.labels["topology.kubernetes.io/zone"], image_version: .metadata.labels["kubernetes.azure.com/node-image-version"],  }'

k scale deployment/inflate --replicas=10 -n inflatens
watch kubectl get po -o wide -n inflatens
kubectl scale deployment/inflate --replicas=0 -n inflatens

```

```
manual scaling: 
================
az aks show --resource-group simple-aks-rg --name simpleaks-cluster1 --query agentPoolProfiles
az aks scale --resource-group simple-aks-rg --name simpleaks --node-count 1 --nodepool-name <your node pool name>
az aks nodepool scale --name apppoolone --cluster-name simpleaks --resource-group simple-aks-rg  --node-count 0

auto-scaling
=============
az aks create --resource-group simple-aks-rg --name simpleaks --node-count 1 --vm-set-type VirtualMachineScaleSets --load-balancer-sku standard --enable-cluster-autoscaler --min-count 1 --max-count 3
az aks update --resource-group simple-aks-rg --name simpleaks --enable-cluster-autoscaler --min-count 1 --max-count 3
----
az aks nodepool update --resource-group simple-aks-rg --cluster-name simpleaks --name nodepool1 --update-cluster-autoscaler --min-count 1 --max-count 5
az aks nodepool update --resource-group simple-aks-rg --cluster-name simpleaks --name nodepool1 --disable-cluster-autoscaler

az aks update --resource-group simple-aks-rg --name simpleaks --cluster-autoscaler-profile scan-interval=30s, scale-down-delay-after-add=0s,scale-down-delay-after-failure=30s,scale-down-unneeded-time=3m,scale-down-unready-time=3m,max-graceful-termination-sec=30,skip-nodes-with
