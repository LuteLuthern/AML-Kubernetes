# Deploy AzureML extension to your Kubernetes cluster

With a simple AzureML extension deployment, an AKS cluster or any Azure Arc enabled Kubernetes cluster, you can instantly onboard data science professionals to submit ML workload to the Kubernetes clusters by using existing Azure ML tools and service capabilities. You can deploy AzureML extension and enables Kubernetes cluster for following machine learning needs:
* Deploy AzureML extension for model training and batch inference
* Deploy AzureML extension for real-time inferencing only
* Deploy AzureML extension for both model training and inferencing

Upon AzureML extension deployment completes, it will create following resources in Azure cloud and in Kubernetes cluster, depending on each AzureML extension deployment scenario:
   |Resource name  |Resource type |Training |Inference |Training and Inference| Description |
   |--|--|--|--|--|--|
   |Azure Service Bus|Azure resource|**&check;**|**&check;**|**&check;**|Used by gateway to sync job and cluster status to Azure Machine Learning services regularly.|
   |Azure Relay|Azure resource|**&check;**|**&check;**|**&check;**|Route traffic from Azure Machine Learning services to the Kubernetes cluster.|
   |aml-operator|Kubernetes deployment|**&check;**|N/A|**&check;**|Manage the lifecycle of training jobs.|
   |{EXTENSION-NAME}-kube-state-metrics|Kubernetes deployment|**&check;**|**&check;**|**&check;**|Export the cluster-related metrics to Prometheus.|
   |{EXTENSION-NAME}-prometheus-operator|Kubernetes deployment|**&check;**|**&check;**|**&check;**| Provide Kubernetes native deployment and management of Prometheus and related monitoring components.|
   |amlarc-identity-controller|Kubernetes deployment|N/A|**&check;**|**&check;**|Request and renew Blob/Azure Container Registry token with managed identity for infrastructure and user containers.|
   |amlarc-identity-proxy|Kubernetes deployment|N/A|**&check;**|**&check;**|Request and renew Blob/Azure Container Registry token with managed identity for infrastructure and user containers.|
   |azureml-fe|Kubernetes deployment|N/A|**&check;**|**&check;**|The front-end component that routes incoming inference requests to deployed services.|
   |inference-operator-controller-manager|Kubernetes deployment|N/A|**&check;**|**&check;**|Manage the lifecycle of inference endpoints. |
   |metrics-controller-manager|Kubernetes deployment|**&check;**|**&check;**|**&check;**|Manage the configuration for Prometheus|
   |relayserver|Kubernetes deployment|**&check;**|**&check;**|**&check;**|Pass the job spec from Azure Machine Learning services to the Kubernetes cluster.|
   |cluster-status-reporter|Kubernetes deployment|**&check;**|**&check;**|**&check;**|Gather the nodes and resource information, and upload it to Azure Machine Learning services.|
   |nfd-master|Kubernetes deployment|**&check;**|N/A|**&check;**|Node feature discovery.|
   |gateway|Kubernetes deployment|**&check;**|**&check;**|**&check;**|Send nodes and cluster resource information to Azure Machine Learning services.|
   |csi-blob-controller|Kubernetes deployment|**&check;**|N/A|**&check;**|Azure Blob Storage Container Storage Interface(CSI) driver.|
   |csi-blob-node|Kubernetes daemonset|**&check;**|N/A|**&check;**|Azure Blob Storage Container Storage Interface(CSI) driver.|
   |fluent-bit|Kubernetes daemonset|**&check;**|**&check;**|**&check;**|Gather infrastructure components' log.|
   |k8s-host-device-plugin-daemonset|Kubernetes daemonset|**&check;**|**&check;**|**&check;**|Expose fuse to pods on each node.|
   |nfd-worker|Kubernetes daemonset|**&check;**|N/A|**&check;**|Node feature discovery.|
   |prometheus-prom-prometheus|Kubernetes statefulset|**&check;**|**&check;**|**&check;**|Gather and send job metrics to Azure.|
   |frameworkcontroller|Kubernetes statefulset|**&check;**|N/A|**&check;**|Manage the lifecycle of Azure Machine Learning training pods.|
   |alertmanager|Kubernetes statefulset|**&check;**|N/A|**&check;**|Handle alerts sent by client applications such as the Prometheus server.|

> **<span style="color:orange">Important**:</span> 
   > * Azure ServiceBus and Azure Relay resources  are under the same resource group as the Arc cluster resource. These resources are used to communicate with the Kubernetes cluster and modifying them will break attached compute targets.
   > * By default, the deployed kubernetes deployment resourses are randomly deployed to 1 or more nodes of the cluster, and daemonset resource are deployed to ALL nodes. If you want to restrict the extension deployment to specific nodes, please use `nodeSelector` configuration setting described as below.


> **<span stype="color:orane">Notes**:</span>
   > * **{EXTENSION-NAME}:** This is the extension name specified with ```az k8s-extension create --name``` CLI command. 

## Review AzureML deployment configuration settings

Use ```k8s-extension create``` CLI command to deploy AzureML extension, review list of required and optional parameters for ```k8s-extension create``` CLI command [here](https://docs.microsoft.com/en-us/cli/azure/k8s-extension?view=azure-cli-latest#az_k8s_extension_create). For AzureML extension deployment configurations, use ```--config``` or ```--config-protected``` to specify list of ```key=value``` pairs. Following is the list of configuration settings available to be used for different AzureML extension deployment scenario ns.
   |Configuration Setting Key Name  |Description  |Training |Inference |Training and Inference
   |--|--|--|--|--|
   |```enableTraining``` |```True``` or ```False```, default ```False```. **Must** be set to ```True``` for AzureML extension deployment with Machine Learning model training support.  |  **&check;**| N/A |  **&check;** |
   |```logAnalyticsWS```  |```True``` or ```False```, default ```False```. AzureML extension integrates with Azure LogAnalytics Workspace to provide log viewing and analysis capability through LogAalytics Workspace. This setting must be explicitly set to ```True``` if customer wants to use this capability. LogAnalytics Workspace cost may apply.  |Optional |Optional |Optional |
   |```installNvidiaDevicePlugin```  | ```True``` or ```False```, default ```True```. Nvidia Device Plugin is required for ML workloads on Nvidia GPU hardware. By default, AzureML extension deployment will install Nvidia Device Plugin regardless Kubernetes cluster has GPU hardware or not. User can specify this configuration setting to False if Nvidia Device Plugin installation is not required (either it is installed already or there is no plan to use GPU for any workload). | Optional |Optional |Optional |
   |```blobCsiDriverEnabled```| ```True``` or ```False```, default ```True```.  Blob CSI driver is required for ML workloads. User can specify this configuration setting to ```False``` if it was installed already. | Optional |Optional |Optional |
   | ```enableInference``` |```True``` or ```False```, default ```False```.  **Must** be set to ```True``` for AzureML extension deployment with Machine Learning inference support. |N/A| **&check;** |  **&check;** |
   | ```allowInsecureConnections``` |```True``` or ```False```, default False. This **must** be set to ```True``` for AzureML extension deployment with HTTP endpoints support for inference, when ```sslCertPemFile``` and ```sslKeyPemFile``` are not provided. |N/A| Optional |  Optional |
   | ```privateEndpointNodeport``` |```True``` or ```False```, default ```False```.  **Must** be set to ```True``` for AzureML deployment with Machine Learning inference private endpoints support using serviceType nodePort. | N/A| Optional |  Optional |
   | ```privateEndpointILB``` |```True``` or ```False```, default ```False```.  **Must** be set to ```True``` for AzureML extension deployment with Machine Learning inference private endpoints support using serviceType internal load balancer | N/A| Optional |  Optional |
   | ```inferenceLoadBalancerHA``` |```True``` or ```False```, default ```True```. By default, AzureML extension will deploy 3 ingress controller replicas for high availability, which requires at least 3 workers in a cluster. Set this to ```False``` if you have fewer than 3 workers and want to deploy AzureML extension for development and testing only, in this case it will deploy one ingress controller replica only. | N/A| Optional |  Optional |
   |```openshift``` | ```True``` or ```False```, default ```False```. Set to ```True``` if you deploy AzureML extension on ARO or OCP cluster. The deployment process will automatically compile a policy package and load policy package on each node so AzureML services operation can function properly.  | Optional| Optional |  Optional |
   |```nodeSelector``` | Set the node selector so the extension components and the training/inference workloads will only be deployed to the nodes with all specified selectors. Usage: `nodeSelector.key=value`, support multiple selectors. Example: `nodeSelector.node-purpose=worker nodeSelector.node-region=eastus`| Optional| Optional |  Optional |
   |```sslCname``` |A SSL CName to use if enabling SSL validation on the cluster.   |  N/A | Optional |  Optional |

   |Configuration Protected Setting Key Name  |Description  |Training |Inference |Training and Inference
   |--|--|--|--|--|
   | ```sslCertPemFile```, ```sslKeyPemFile``` |Path to SSL certificate and key file (PEM-encoded), required for AzureML extension deployment with HTTPS endpoint support for inference, when  ``allowInsecureConnections`` is set to False. | N/A| Optional |  Optional |


## Prerequisites 

-  For AzureML extension deployment on ARO or OCP cluster, grant privileged access to AzureML service accounts, run ```oc edit scc privileged``` command, and add following service accounts under "users:":

   * ```system:serviceaccount:azure-arc:azure-arc-kube-aad-proxy-sa```
   * ```system:serviceaccount:azureml:{EXTENSION-NAME}-kube-state-metrics``` 
   * ```system:serviceaccount:azureml:cluster-status-reporter```
   * ```system:serviceaccount:azureml:prom-admission```
   * ```system:serviceaccount:azureml:default```
   * ```system:serviceaccount:azureml:prom-operator```
   * ```system:serviceaccount:azureml:csi-blob-node-sa```
   * ```system:serviceaccount:azureml:csi-blob-controller-sa```
   * ```system:serviceaccount:azureml:load-amlarc-selinux-policy-sa```
   * ```system:serviceaccount:azureml:azureml-fe```
   * ```system:serviceaccount:azureml:prom-prometheus```
   * ```system:serviceaccount:{KUBERNETES-COMPUTE-NAMESPACE}:default```
   > **<span stype="color:yellow">Notes</span>**
      >* **{EXTENSION-NAME}:** is the extension name specified with ```az k8s-extension create --name``` CLI command. 
      >* **{KUBERNETES-COMPUTE-NAMESPACE}:** is the namespace of kubernetes compute specified with ```az ml compute attach --namespace``` CLI command. Skip configuring 'system:serviceaccount:{KUBERNETES-COMPUTE-NAMESPACE}:default' if no namespace specified with ```az ml compute attach ``` CLI command.

-  Log in to Azure

   ```azurecli
   az login
   az account set --subscription <your-subscription-id>
   ```
   
- If you use Azure Kubernetes Services(AKS) cluster and it's not connected to Azure Arc, please register below feature for your subscription.
  ```azurecli
  az feature register --namespace Microsoft.ContainerService -n AKS-ExtensionManager
  ```

## Deploy AzureML extension for model training or batch inference workload

Following CLI command will deploy AzureML extension and enable Kubernetes cluster for model training and batch inference workload:
   ```azurecli
   az k8s-extension create --name arcml-extension --extension-type Microsoft.AzureML.Kubernetes --config enableTraining=True  --cluster-type connectedClusters --cluster-name <your-connected-cluster-name> --resource-group <resource-group> --scope cluster --auto-upgrade-minor-version False
   ```
> **<span stype="color:orane">Notes**:</span>
   > * **If you deploy AzureML extension on AKS directly without Azure Arc connection, please change ```--cluster-type``` parameter value to ```managedClusters```**

## Deploy AzureML extension for real-time inference workload only

Depending on your network setup, Kubernetes distribution variant, and where your Kubernetes cluster is hosted (in cloud or on-premises), you can choose one of following options to deploy AzureML extension.

> **<span stype="color:orane">Notes**:</span>
   > * **If you deploy AzureML extension on AKS directly without Azure Arc connection, please change ```--cluster-type``` parameter value to ```managedClusters```**

   * **Public HTTPS endpoints support with public load balancer**

      ```azurecli
      az k8s-extension create --name arcml-extension --extension-type Microsoft.AzureML.Kubernetes --cluster-type connectedClusters --cluster-name <your-connected-cluster-name> --config enableInference=True sslCname=<cname> --config-protected sslCertPemFile=<path-to-the-SSL-cert-PEM-file> sslKeyPemFile=<path-to-the-SSL-key-PEM-file> --resource-group <resource-group> --scope cluster --auto-upgrade-minor-version False
      ```
   * **Private HTTPS endpoints support with internal load balancer**   
      ```azurecli
      az k8s-extension create --name amlarc-compute --extension-type Microsoft.AzureML.Kubernetes --cluster-type connectedClusters --cluster-name <your-connected-cluster-name> --config enableInference=True privateEndpointILB=True sslCname=<cname> --config-protected sslCertPemFile=<path-to-the-SSL-cert-PEM-file> sslKeyPemFile=<path-to-the-SSL-key-PEM-file> --resource-group <resource-group> --scope cluster --auto-upgrade-minor-version False
      ```
   * **Private HTTPS endpoints support with NodePort**
      ```azurecli
      az k8s-extension create --name arcml-extension --extension-type Microsoft.AzureML.Kubernetes --cluster-type connectedClusters --cluster-name <your-connected-cluster-name> --resource-group <resource-group> --scope cluster --config enableInference=True privateEndpointNodeport=True sslCname=<cname> --config-protected sslCertPemFile=<path-to-the-SSL-cert-PEM-file> sslKeyPemFile=<path-to-the-SSL-key-PEM-file> --auto-upgrade-minor-version False
      ```  
     > **Note:**
     * Using a NodePort gives you the freedom to set up your own load-balancing solution, to configure environments that are not fully supported by Kubernetes, or even to expose one or more nodes' IPs directly.
     * When you deploy with NodePort service, the scoring url (or swagger url) will be responded with one of Node IP (for example, ```https://<NodeIP>:<NodePort>/<scoring_path>```) and remain unchanged even if the Node is unavailable. But you can replace it with any other Node IP.
   * **Private HTTP endpoints support with internal load balancer**   
      ```azurecli
      az k8s-extension create --name arcml-extension --extension-type Microsoft.AzureML.Kubernetes --cluster-type connectedClusters --cluster-name <your-connected-cluster-name> --config enableInference=True privateEndpointILB=True allowInsecureConnections=True --resource-group <resource-group> --scope cluster --auto-upgrade-minor-version False
      ```
   * **Private HTTP endpoints support with NodePort**
      ```azurecli
      az k8s-extension create --name arcml-extension --extension-type Microsoft.AzureML.Kubernetes --cluster-type connectedClusters --cluster-name <your-connected-cluster-name> --config enableInference=True privateEndpointNodeport=True allowInsecureConnections=Ture --resource-group <resource-group> --scope cluster --auto-upgrade-minor-version False
      ```
   * **Public HTTP endpoints support with public load balancer - the least secure way, NOT recommended**
      ```azurecli
      az k8s-extension create --name arcml-extension --extension-type Microsoft.AzureML.Kubernetes --cluster-type connectedClusters --cluster-name <your-connected-cluster-name>  --config enableInference=True allowInsecureConnections=True --resource-group <resource-group> --scope cluster --auto-upgrade-minor-version False
      ```
## Deploy AzureML extension for training, batch inference, and real-time inference workload

To enable Kubernetes cluster for all kinds of ML workload, choose one of above inference deployment options and append config settings for training and batch inference. Following CLI command will enable cluster with real-time inference HTTPS endpoints support, training, and batch inference workload:
   ```azurecli
   az k8s-extension create --name arcml-extension --extension-type Microsoft.AzureML.Kubernetes --cluster-type connectedClusters --cluster-name <your-connected-cluster-name> --config enableTraining=True enableInference=True --config-protected sslCertPemFile=<path-to-the-SSL-cert-PEM-file> sslKeyPemFile=<path-to-the-SSL-key-PEM-file> --resource-group <resource-group> --scope cluster --auto-upgrade-minor-version False
   ```

> **<span stype="color:orane">Notes**:</span>
   > * **If you deploy AzureML extension on AKS directly without Azure Arc connection, please change ```--cluster-type``` parameter value to ```managedClusters```**

## Verify your AzureML extension deployment

1. Run the following CLI command to check AzureML extension details:

   ```azurecli
   az k8s-extension show --name arcml-extension --cluster-type connectedClusters --cluster-name <your-connected-cluster-name> --resource-group <resource-group>
   ```

1. In the response, look for "name": "arcml-extension" and "provisioningState": "Succeeded". Note it might show "provisioningState": "Pending" for the first few minutes.

1. If the provisioningState shows Succeeded, run the following command on your machine with the kubeconfig file pointed to your cluster to check that all pods under "azureml" namespace are in 'Running' state:

   ```bash
    kubectl get pods -n azureml
   ```
## Update Azure Machine Learning extension

Use ```k8s-extension update``` CLI command to update the mutable properties of  AzureML extension, review list of required and optional parameters for ```k8s-extension update``` CLI command [here](https://docs.microsoft.com/en-us/cli/azure/k8s-extension?view=azure-cli-latest#az_k8s_extension_update). 

1.	Azure Arc supports update of  ``--auto-upgrade-minor-version``, ``--version``,  ``--config``, ``--config-protected``.  
2.	For configurationSettings, only the settings that require update need to be provided. If the user provides all settings, they would be merged/overwritten with the provided values. 
3.	For ConfigurationProtectedSettings, ALL  settings should be provided. If some settings are omitted, those settings would be considered obsolete and deleted. 

> [!IMPORTANT]
> **Don't** update following configs if you have active training workloads or real-time inference endpoints. Otherwise, the training jobs will be impacted and endpoints unavailable.
> 
> * `enableTraining` from `True` to `False`
> * `installNvidiaDevicePlugin` from `True` to `False` when using GPU.
> * `nodeSelector`. The update operation can't remove existing nodeSelectors. It can only update existing ones or add new ones.
>
> **Don't** update following configs if you have active real-time inference endpoints, otherwise, the endpoints will be unavailable.
> * `allowInsecureConnections`
> * `privateEndpointNodeport`
> * `privateEndpointILB`
> *  To update `logAnalyticsWS` from `True` to `False`, provide all original `configurationProtectedSettings`. Otherwise, those settings are considered obsolete and deleted.

## Delete Azure Machine Learning extension

Use [``k8s-extension delete``](https://docs.microsoft.com/en-us/cli/azure/k8s-extension?view=azure-cli-latest#az_k8s_extension_delete) CLI command to delete the existed AzureMl extension.

It takes around 10 minutes to delete all components deployed to the Kubernetes cluster. Run `kubectl get pods -n azureml` to check if all components were deleted.
