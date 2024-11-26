# workloadmanager
Workload Manager is used move AKS workloads between different node pools using affinity. 

## Description
Some applications cannot take full advantage of the Kubernetes high-availability concepts. This CRD has been designed to manage these workloads. 

Based on the YAML you provide, the affinity will be adjusted to change the provided key and value. Kubernetes will then re-schedule the workload based on the updated configuration.

It can be thought of as a blue/green deployment pattern, but all within a single AKS cluster (for example 2 node pools, one green, one blue)

## Getting Started

### Prerequisites
- Access to a Azure Kubernetes Service 1.26+
- One or more Deployments or Statefulsets which are scheduled with node affinity 
- A Managed Identity with AKS Contributor Role for the cluster

### Deployment
Helm chart:
```
kubectl create ns operations
helm repo add k8smanagers https://k8smanagers.blob.core.windows.net/helm/
helm install workloadmanager k8smanagers/workloadmanager -n operations
```
To customize the values, create a values.yaml and including with -f on helm command. Supported

* private image repository
* custom image tag
* custom image pull secrets
* custom labels for Deployment
* additional environment variables (see security section below)

Examples:

```yaml
controllerManager:
  deployment:
    customLabels:                                # Custom labels
      custom1: value1
      custom2: value2
  manager:
    repository: my-image-repo/workload-manager   # Private image repo
    tag: specific-tag                            # Custom image tag
imagePullSecrets:
  - name: my-image-pull-secret                   # Custom image pull secret
```

#### Verification
```
helm list --filter 'workloadmanager' 
kubectl get crd workloadmanagers.k8smanagers.greyridge.com
```

### Configuration
To activate the CRD configuration, apply a YAML

```yaml
apiVersion: k8smanagers.greyridge.com/v1
kind: WorkloadManager
metadata:
  labels:
    app.kubernetes.io/name: workloadmanager
    app.kubernetes.io/managed-by: kustomize
  name: workloadmanager-sample
spec:
  # Subscription ID of the cluster
  subscriptionId: "3e54eb54-xxxx-yyyy-zzzz-d7b190cd45cf" 
  
  # Resource group containing the cluster
  resourceGroup: "node-upgrader"
  
  # The name of the cluster
  clusterName: "lm-cluster"
  
  # When this flag is true, if an error occurs, the CRD will ask Kubernetes to retry it automatically
  # This will result in the CRD being re-executed with the current configuration.
  # When this flag is false, the CRD will only be executed when the configuration is new/changed.
  retryOnError: false
  
  # When this flag is true, no changes will be executed, only logged
  testMode: false
  
  # A list of procedures (a list of workloads to move). Will be executed in order
  procedures:
      # A human-friendly description 
    - description: "move-services"
      # The type of resource to change: deployment|statefulset
      type: "deployment"
      # The namespace that contains the resource
      namespace: "my-app-ns"
      # A list of the resources to change. Will be executed in order. Execution will wait until the workload is ready or timeout occurs.
      workloads:
        - "app1"
        - "app2"
        - "app3"
      # The node affinity to change.
      #    key: The key/label
      #    initial: The existing value of this key (used for verification/rollback)
      #    target: The target value for this key
      affinity:
        key: "agentpool"
        initial: "servicesblue"
        target: "servicesglas"
      # The time, in seconds, to wait for the pods to activate after changing the node affinity.
      # If the pod doesn't activate within this time period, it will move to the next 
      # workload in the list (the pod might well activate in time). If not provided, defaults to 600
      timeout: 600

```

## Details

### Affinity

The CRD assumes you have node affinity for _statefulset_ and/or _deployment_ like this (key and value are user-defined)

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: agentpool
              operator: In
              values:
                - servicesblue
```

### Security
The controller users DefaultAzureCredential. The DefaultAzureCredential is a feature of the Azure Identity library that simplifies
authentication for applications running on Azure. It automatically selects the best available credential based on the environment
it's running in, making it easier to manage authentication across different stages of development and deployment.

The controller has been tested with
* ManagedIdentityCredential
* EnvironmentCredential (Enterprise app/Service Principal)

#### Service Principal
The SPN details can be set when installing the helm chart. Create a values.yaml and include with -f on helm command.

Example:

```kubectl create secret generic azure-client-secret --from-literal=AZURE_CLIENT_SECRET=myclientsecret -n operations ```

```yaml
controllerManager:
  deployment:
    env:
      - name: AZURE_CLIENT_ID
        value: aaa-bbb-ccc
      - name: AZURE_TENANT_ID
        value: xxx-yyy-zzz
      - name: AZURE_CLIENT_SECRET
        valueFrom:
          secretKeyRef:
            name: azure-client-secret
            key: AZURE_CLIENT_SECRET
```

Assign the role _Azure Kubernetes Service Contributor Role_ to the MI which matches the details above.

#### Managed Identity
During AKS provisioning, a "managed cluster" resource group will be created, named `MC_[rgroup]_[clustername]_[region]`.
This resource group will contain a managed identity, usually named `[clustername]_agentpool`.
Add an Azure role assignment to this MI. The role to add is _Azure Kubernetes Service Contributor Role_.


### Contributions
This controller has been written using kubebuilder https://book.kubebuilder.io

I would love to have some contributors to the project. Contact me on github

### Uninstall
Uninstall the helm chart
```
helm uninstall -n operations workloadmanager
```

## License

Copyright 2024.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.

## Appendix
### Role Permissions
The following Action permissions are required
```
"Microsoft.ContainerService/managedClusters/agentPools/read",
"Microsoft.ContainerService/managedClusters/agentPools/write",
"Microsoft.ContainerService/managedClusters/read",
"Microsoft.ContainerService/managedClusters/diagnosticsState/read",
"Microsoft.ContainerService/managedClusters/agentPools/upgradeProfiles/read",
"Microsoft.ContainerService/managedClusters/upgradeProfiles/read",
"Microsoft.OperationalInsights/workspaces/sharedkeys/read",
"Microsoft.OperationalInsights/workspaces/read",
"Microsoft.OperationalInsights/workspaces/write",
"Microsoft.OperationsManagement/solutions/write",
"Microsoft.OperationsManagement/solutions/read",
"Microsoft.ContainerService/managedClusters/agentPools/delete",
"Microsoft.ContainerService/managedClusters/listClusterUserCredential/action"
```

### Debugging
Set the DEBUG_LOGGING to "1" in the env section.

