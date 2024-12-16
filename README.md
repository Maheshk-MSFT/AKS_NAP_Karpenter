

```
===================================================
# (1) STEPS FOR ENABLING AKS NODE AUTO-PROVISIONING
===================================================
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

---

k get nodes -o json | jq '.items[] | {name: .metadata.name, instance_type: .metadata.labels["node.kubernetes.io/instance-type"], nodepool_type: .metadata.labels["kubernetes.azure.com/nodepool-type"], karpeneter_nodepool: .metadata.labels ["karpenter.sh/nodedpool"], 
topology_spread: .metadata.labels["topology.kubernetes.io/zone"], karpeneter_nodepool: .metadata.labels ["karpenter.sh/capacity-type"],image_version: .metadata.labels["kubernetes.azure.com/node-image-version"],  }'

---

k get no -o wide

k get NodePool default -o yaml
k edit aksnodeclass default

k apply -f workload.yaml (refer this repo)
---

# you can scale/generate load test to see the karpenter events
k scale deploy -n global-azure-ns --replicas=20 --all

# keep this in separate window to see it in action
k get events -A --field-selector source=karpenter -w

# we should see addtional nodes coming in
k get no -o wide

# run this command to see new nodes of diff size coming from D Series

k get nodes -o json | jq '.items[] | {name: .metadata.name, instance_type: .metadata.labels["node.kubernetes.io/instance-type"], nodepool_type: .metadata.labels["kubernetes.azure.com/nodepool-type"], karpeneter_nodepool: .metadata.labels ["karpenter.sh/nodedpool"], 
topology_spread: .metadata.labels["topology.kubernetes.io/zone"], karpeneter_nodepool: .metadata.labels ["karpenter.sh/capacity-type"],image_version: .metadata.labels["kubernetes.azure.com/node-image-version"],  }'

```
<img width="956" alt="nap3" src="https://github.com/Maheshk-MSFT/AKS_NAP_Karpenter/assets/61469290/aa366992-b51f-443b-9b16-b7cfb75416a8">

<img width="409" alt="nap4" src="https://github.com/Maheshk-MSFT/AKS_NAP_Karpenter/assets/61469290/7a8ce13a-ad5c-4ab8-8c00-866cbb341e30">

<img width="948" alt="nap11" src="https://github.com/Maheshk-MSFT/AKS_NAP_Karpenter/assets/61469290/ac060d76-5ca4-4e38-b3b3-c4a843aa4e5f">

#additional commands
kubectl config get-contexts

```
# (2) STEPS FOR Cluster AutoScaler
====================================================================  
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

az aks create --resource-group simple-aks-rg --name simpleaks --node-count 1 --vm-set-type VirtualMachineScaleSets --load-balancer-sku standard --enable-cluster-autoscaler --min-count 1 --max-count 3
az aks update --resource-group simple-aks-rg --name simpleaks --enable-cluster-autoscaler --min-count 1 --max-count 3
----
az aks nodepool update --resource-group simple-aks-rg --cluster-name simpleaks --name nodepool1 --update-cluster-autoscaler --min-count 1 --max-count 5
az aks nodepool update --resource-group simple-aks-rg --cluster-name simpleaks --name nodepool1 --disable-cluster-autoscaler

az aks update --resource-group simple-aks-rg --name simpleaks --cluster-autoscaler-profile scan-interval=30s, scale-down-delay-after-add=0s,scale-down-delay-after-failure=30s,scale-down-unneeded-time=3m,scale-down-unready-time=3m,max-graceful-termination-sec=30,skip-nodes-with
---
====================================================================  
# (3) STEPS FOR Manual Scaling
====================================================================  
az aks show --resource-group simple-aks-rg --name simpleaks-cluster1 --query agentPoolProfiles
az aks scale --resource-group simple-aks-rg --name simpleaks --node-count 1 --nodepool-name <your node pool name>
az aks nodepool scale --name apppoolone --cluster-name simpleaks --resource-group simple-aks-rg  --node-count 0

-----------------------
UPDATE:Dec/16/2024

```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: rabbitmq
  namespace: karpdemo
spec:
  serviceName: rabbitmq
  replicas: 15
  selector:
    matchLabels:
      app: rabbitmq
  template:
    metadata:
      labels:
        app: rabbitmq
    spec:
      nodeSelector:
        "kubernetes.io/os": linux
      containers:
      - name: rabbitmq
        image: mcr.microsoft.com/mirror/docker/library/rabbitmq:3.10-management-alpine
        ports:
        - containerPort: 5672
          name: rabbitmq-amqp
        - containerPort: 15672
          name: rabbitmq-http
        env:
        - name: RABBITMQ_DEFAULT_USER
          value: "username"
        - name: RABBITMQ_DEFAULT_PASS
          value: "password"
        resources:
          requests:
            cpu: "1"
            memory: "1Gi"
          limits:
            cpu: "1"
            memory: "1Gi"
        volumeMounts:
        - name: rabbitmq-enabled-plugins
          mountPath: /etc/rabbitmq/enabled_plugins
          subPath: enabled_plugins
      volumes:
      - name: rabbitmq-enabled-plugins
        configMap:
          name: rabbitmq-enabled-plugins
          items:
          - key: rabbitmq_enabled_plugins
            path: enabled_plugins
---
apiVersion: v1
data:
  rabbitmq_enabled_plugins: |
    [rabbitmq_management,rabbitmq_prometheus,rabbitmq_amqp1_0].
kind: ConfigMap
metadata:
  name: rabbitmq-enabled-plugins   
  namespace: karpdemo  
---
apiVersion: v1
kind: Service
metadata:
  name: rabbitmq
  namespace: karpdemo
spec:
  selector:
    app: rabbitmq
  ports:
    - name: rabbitmq-amqp
      port: 5672
      targetPort: 5672
    - name: rabbitmq-http
      port: 15672
      targetPort: 15672
  type: ClusterIP
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: karpdemo
spec:
  replicas: 15
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      nodeSelector:
        "kubernetes.io/os": linux
      containers:
      - name: order-service
        image: ghcr.io/azure-samples/aks-store-demo/order-service:latest
        ports:
        - containerPort: 3000
        env:
        - name: ORDER_QUEUE_HOSTNAME
          value: "rabbitmq"
        - name: ORDER_QUEUE_PORT
          value: "5672"
        - name: ORDER_QUEUE_USERNAME
          value: "username"
        - name: ORDER_QUEUE_PASSWORD
          value: "password"
        - name: ORDER_QUEUE_NAME
          value: "orders"
        - name: FASTIFY_ADDRESS
          value: "0.0.0.0"
        resources:
          requests:
            cpu: "1"
            memory: "500Mi"
          limits:
            cpu: "2"
            memory: "1Gi"
        startupProbe:
          httpGet:
            path: /health
            port: 3000
          failureThreshold: 5
          initialDelaySeconds: 20
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 3000
          failureThreshold: 3
          initialDelaySeconds: 3
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          failureThreshold: 5
          initialDelaySeconds: 3
          periodSeconds: 3
      initContainers:
      - name: wait-for-rabbitmq
        image: busybox
        command: ['sh', '-c', 'until nc -zv rabbitmq 5672; do echo waiting for rabbitmq; sleep 2; done;']
        resources:
          requests:
            cpu: "1"
            memory: 50Mi
          limits:
            cpu: "1"
            memory: 128Mi    
---
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: karpdemo
spec:
  type: ClusterIP
  ports:
  - name: http
    port: 3000
    targetPort: 3000
  selector:
    app: order-service
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-service
  namespace: karpdemo
spec:
  replicas: 15
  selector:
    matchLabels:
      app: product-service
  template:
    metadata:
      labels:
        app: product-service
    spec:
      nodeSelector:
        "kubernetes.io/os": linux
      containers:
      - name: product-service
        image: ghcr.io/azure-samples/aks-store-demo/product-service:latest
        ports:
        - containerPort: 3002
        env: 
        - name: AI_SERVICE_URL
          value: "http://ai-service:5001/"
        resources:
          requests:
            cpu: "1"
            memory: "1Gi"
          limits:
            cpu: "1"
            memory: "1Gi"
        readinessProbe:
          httpGet:
            path: /health
            port: 3002
          failureThreshold: 3
          initialDelaySeconds: 3
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /health
            port: 3002
          failureThreshold: 5
          initialDelaySeconds: 3
          periodSeconds: 3
---
apiVersion: v1
kind: Service
metadata:
  name: product-service
  namespace: karpdemo
spec:
  type: ClusterIP
  ports:
  - name: http
    port: 3002
    targetPort: 3002
  selector:
    app: product-service
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: store-front
  namespace: karpdemo
spec:
  replicas: 15
  selector:
    matchLabels:
      app: store-front
  template:
    metadata:
      labels:
        app: store-front
    spec:
      nodeSelector:
        "kubernetes.io/os": linux
      containers:
      - name: store-front
        image: ghcr.io/azure-samples/aks-store-demo/store-front:latest
        ports:
        - containerPort: 8080
          name: store-front
        env: 
        - name: VUE_APP_ORDER_SERVICE_URL
          value: "http://order-service:3000/"
        - name: VUE_APP_PRODUCT_SERVICE_URL
          value: "http://product-service:3002/"
        resources:
          requests:
            cpu: "1"
            memory: 200Mi
          limits:
            cpu: "1"
            memory: 512Mi
        startupProbe:
          httpGet:
            path: /health
            port: 8080
          failureThreshold: 3
          initialDelaySeconds: 5
          periodSeconds: 5
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          failureThreshold: 3
          initialDelaySeconds: 3
          periodSeconds: 3
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          failureThreshold: 5
          initialDelaySeconds: 3
          periodSeconds: 3
---
apiVersion: v1
kind: Service
metadata:
  name: store-front
  namespace: karpdemo
spec:
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: store-front
  type: LoadBalancer
```

```
new cluster (cilium n/w plane is pre-req) 
az aks create --name aksclustername--resource-group rgname--node-provisioning-mode Auto --network-plugin azure --network-plugin-mode overlay --network-dataplane cilium

Update existing,
az aks update --name mikkykarp --resource-group aks-cilium-rg --node-provisioning-mode Auto --network-plugin azure --network-plugin-mode overlay --network-dataplane cilium

verify enablement, 
kubectl api-resources | grep -e aksnodeclasses -e nodeclaims -e nodepools

az aks update --name mikkykarp --resource-group aks-cilium-rg --disable-cluster-autoscaler


k get nodes -o json | jq '.items[] | {name: .metadata.name, instance_type: .metadata.labels["node.kubernetes.io/instance-type"], nodepool_type: .metadata.labels["kubernetes.azure.com/nodepool-type"], karpeneter_nodepool: .metadata.labels ["karpenter.sh/nodedpool"], 
topology_spread: .metadata.labels["topology.kubernetes.io/zone"], karpeneter_nodepool: .metadata.labels ["karpenter.sh/capacity-type"],image_version: .metadata.labels["kubernetes.azure.com/node-image-version"],  }'

```
NOTE: You will not see the existing nodepool scaling.
You will see new VM's getting added to the MG_resource group - standalone VM's with various SKu size.

<img width="943" alt="image" src="https://github.com/user-attachments/assets/24a869c3-cb7c-4c93-9dba-eba29338813e" />
 
