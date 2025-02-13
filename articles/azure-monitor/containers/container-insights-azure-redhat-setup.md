---
title: Configure Azure Red Hat OpenShift v3.x with Container insights | Microsoft Docs
description: This article describes how to configure monitoring of a Kubernetes cluster with Azure Monitor hosted on Azure Red Hat OpenShift version 3 and higher.
ms.topic: conceptual
ms.date: 06/30/2020
---

# Configure Azure Red Hat OpenShift v3 with Container insights

>[!IMPORTANT]
> Azure Red Hat OpenShift 3.11 will be retired June 2022.
>
> As of October 2020 you will no longer be able to create new 3.11 clusters.
> Existing 3.11 clusters will continue to operate until June 2022 but will no be longer supported after that date.
>
> Follow this guide to [create an Azure Red Hat OpenShift 4 cluster](../../openshift/tutorial-create-cluster.md).
> If you have specific questions, [please contact us](mailto:aro-feedback@microsoft.com).

Container insights provides rich monitoring experience for the Azure Kubernetes Service (AKS) and AKS Engine clusters. This article describes how to enable monitoring of Kubernetes clusters hosted on [Azure Red Hat OpenShift](../../openshift/intro-openshift.md) version 3 and latest supported version of version 3, to achieve a similar monitoring experience.

>[!NOTE]
>Support for Azure Red Hat OpenShift is a feature in public preview at this time.
>

Container insights can be enabled for new, or one or more existing deployments of Azure Red Hat OpenShift using the following supported methods:

- For an existing cluster from the Azure portal or using Azure Resource Manager template.
- For a new cluster using Azure Resource Manager template, or while creating a new cluster using the [Azure CLI](/cli/azure/openshift#az-openshift-create).

## Supported and unsupported features

Container insights supports monitoring Azure Red Hat OpenShift as described in the [Overview](container-insights-overview.md) article, except for the following features:

- Live Data (preview)
- [Collect metrics](container-insights-update-metrics.md) from cluster nodes and pods and storing them in the Azure Monitor metrics database

## Prerequisites

- A [Log Analytics workspace](../logs/workspace-design.md).

    Container insights supports a Log Analytics workspace in the regions listed in Azure [Products by region](https://azure.microsoft.com/global-infrastructure/services/?regions=all&products=monitor). To create your own workspace, it can be created through [Azure Resource Manager](../logs/resource-manager-workspace.md), through [PowerShell](../logs/powershell-workspace-configuration.md?toc=%2fpowershell%2fmodule%2ftoc.json), or in the [Azure portal](../logs/quick-create-workspace.md).

- To enable and access the features in Container insights, at a minimum you need to be a member of the Azure *Contributor* role in the Azure subscription, and a member of the [*Log Analytics Contributor*](../logs/manage-access.md#azure-rbac) role of the Log Analytics workspace configured with Container insights.

- To view the monitoring data, you are a member of the [*Log Analytics reader*](../logs/manage-access.md#azure-rbac) role permission with the Log Analytics workspace configured with Container insights.

## Identify your Log Analytics workspace ID

 To integrate with an existing Log Analytics workspace, start by identifying the full resource ID of your Log Analytics workspace. The resource ID of the workspace is required for the parameter `workspaceResourceId` when you enable monitoring using the Azure Resource Manager template method.

1. List all the subscriptions that you have access to by running the following command:

    ```azurecli
    az account list --all -o table
    ```

    The output will look like the following:

    ```azurecli
    Name                                  CloudName    SubscriptionId                        State    IsDefault
    ------------------------------------  -----------  ------------------------------------  -------  -----------
    Microsoft Azure                       AzureCloud   0fb60ef2-03cc-4290-b595-e71108e8f4ce  Enabled  True
    ```

1. Copy the value for **SubscriptionId**.

1. Switch to the subscription that hosts the Log Analytics workspace by running the following command:

    ```azurecli
    az account set -s <subscriptionId of the workspace>
    ```

1. Display the list of workspaces in your subscriptions in the default JSON format by running the following command:

    ```
    az resource list --resource-type Microsoft.OperationalInsights/workspaces -o json
    ```

1. In the output, find the workspace name, and then copy the full resource ID of that Log Analytics workspace under the field **ID**.

## Enable for a new cluster using an Azure Resource Manager template

Perform the following steps to deploy an Azure Red Hat OpenShift cluster with monitoring enabled. Before proceeding, review the tutorial [Create an Azure Red Hat OpenShift cluster](../../openshift/tutorial-create-cluster.md) to understand the dependencies that you need to configure so your environment is set up correctly.

This method includes two JSON templates. One template specifies the configuration to deploy the cluster with monitoring enabled, and the other contains parameter values that you configure to specify the following:

- The Azure Red Hat OpenShift cluster resource ID.

- The resource group the cluster is deployed in.

- [Azure Active Directory tenant ID](../../openshift/howto-create-tenant.md#create-a-new-azure-ad-tenant) noted after performing the steps to create one or one already created.

- [Azure Active Directory client application ID](../../openshift/howto-aad-app-configuration.md#create-an-azure-ad-app-registration) noted after performing the steps to create one or one already created.

- [Azure Active Directory Client secret](../../openshift/howto-aad-app-configuration.md#create-a-client-secret) noted after performing the steps to create one or one already created.

- [Azure AD security group](../../openshift/howto-aad-app-configuration.md#create-an-azure-ad-security-group) noted after performing the steps to create one or one already created.

- Resource ID of an existing Log Analytics workspace. See [Identify your Log Analytics workspace ID](#identify-your-log-analytics-workspace-id) to learn how to get this information.

- The number of master nodes to create in the cluster.

- The number of compute nodes in the agent pool profile.

- The number of infrastructure nodes in the agent pool profile.

If you are unfamiliar with the concept of deploying resources by using a template, see:

- [Deploy resources with Resource Manager templates and Azure PowerShell](../../azure-resource-manager/templates/deploy-powershell.md)

- [Deploy resources with Resource Manager templates and the Azure CLI](../../azure-resource-manager/templates/deploy-cli.md)

If you choose to use the Azure CLI, you first need to install and use the CLI locally. You must be running the Azure CLI version 2.0.65 or later. To identify your version, run `az --version`. If you need to install or upgrade the Azure CLI, see [Install the Azure CLI](/cli/azure/install-azure-cli).

1. Download and save to a local folder, the Azure Resource Manager template and parameter file, to create a cluster with the monitoring add-on using the following commands:

    `curl -LO https://raw.githubusercontent.com/microsoft/Docker-Provider/ci_dev/scripts/onboarding/aro/enable_monitoring_to_new_cluster/newClusterWithMonitoring.json`

    `curl -LO https://raw.githubusercontent.com/microsoft/Docker-Provider/ci_dev/scripts/onboarding/aro/enable_monitoring_to_new_cluster/newClusterWithMonitoringParam.json`

2. Sign in to Azure

    ```azurecli
    az login
    ```

    If you have access to multiple subscriptions, run `az account set -s {subscription ID}` replacing `{subscription ID}` with the subscription you want to use.

3. Create a resource group for your cluster if you don't already have one. For a list of Azure regions that supports OpenShift on Azure, see [Supported Regions](../../openshift/supported-resources.md#azure-regions).

    ```azurecli
    az group create -g <clusterResourceGroup> -l <location>
    ```

4. Edit the JSON parameter file **newClusterWithMonitoringParam.json** and update the following values:

    - *location*
    - *clusterName*
    - *aadTenantId*
    - *aadClientId*
    - *aadClientSecret*
    - *aadCustomerAdminGroupId*
    - *workspaceResourceId*
    - *masterNodeCount*
    - *computeNodeCount*
    - *infraNodeCount*

5. The following step deploys the cluster with monitoring enabled by using the Azure CLI.

    ```azurecli
    az deployment group create --resource-group <ClusterResourceGroupName> --template-file ./newClusterWithMonitoring.json --parameters @./newClusterWithMonitoringParam.json
    ```

    The output resembles the following:

    ```output
    provisioningState       : Succeeded
    ```

## Enable for an existing cluster

Perform the following steps to enable monitoring of an Azure Red Hat OpenShift cluster deployed in Azure. You can accomplish this from the Azure portal or using the provided templates.

### From the Azure portal

1. Sign in to the [Azure portal](https://portal.azure.com).

2. On the Azure portal menu or from the Home page, select **Azure Monitor**. Under the **Insights** section, select **Containers**.

3. On the **Monitor - containers** page, select **Non-monitored clusters**.

4. From the list of non-monitored clusters, find the cluster in the list and click **Enable**. You can identify the results in the list by looking for the value **ARO** under the column **CLUSTER TYPE**.

5. On the **Onboarding to Container insights** page, if you have an existing Log Analytics workspace in the same subscription as the cluster, select it from the drop-down list.  
    The list preselects the default workspace and location that the cluster is deployed to in the subscription.

    ![Enable monitoring for non-monitored clusters](./media/container-insights-onboard/kubernetes-onboard-brownfield-01.png)

    >[!NOTE]
    >If you want to create a new Log Analytics workspace for storing the monitoring data from the cluster, follow the instructions in [Create a Log Analytics workspace](../logs/quick-create-workspace.md). Be sure to create the workspace in the same subscription that the RedHat OpenShift cluster is deployed to.

After you've enabled monitoring, it might take about 15 minutes before you can view health metrics for the cluster.

### Enable using an Azure Resource Manager template

This method includes two JSON templates. One template specifies the configuration to enable monitoring, and the other contains parameter values that you configure to specify the following:

- The Azure RedHat OpenShift cluster resource ID.

- The resource group the cluster is deployed in.

- A Log Analytics workspace. See [Identify your Log Analytics workspace ID](#identify-your-log-analytics-workspace-id) to learn how to get this information.

If you are unfamiliar with the concept of deploying resources by using a template, see:

- [Deploy resources with Resource Manager templates and Azure PowerShell](../../azure-resource-manager/templates/deploy-powershell.md)

- [Deploy resources with Resource Manager templates and the Azure CLI](../../azure-resource-manager/templates/deploy-cli.md)

If you choose to use the Azure CLI, you first need to install and use the CLI locally. You must be running the Azure CLI version 2.0.65 or later. To identify your version, run `az --version`. If you need to install or upgrade the Azure CLI, see [Install the Azure CLI](/cli/azure/install-azure-cli).

1. Download the template and parameter file to update your cluster with the monitoring add-on using the following commands:

    `curl -LO https://raw.githubusercontent.com/microsoft/Docker-Provider/ci_dev/scripts/onboarding/aro/enable_monitoring_to_existing_cluster/existingClusterOnboarding.json`

    `curl -LO https://raw.githubusercontent.com/microsoft/Docker-Provider/ci_dev/scripts/onboarding/aro/enable_monitoring_to_existing_cluster/existingClusterParam.json`

2. Sign in to Azure

    ```azurecli
    az login
    ```

    If you have access to multiple subscriptions, run `az account set -s {subscription ID}` replacing `{subscription ID}` with the subscription you want to use.

3. Specify the subscription of the Azure RedHat OpenShift cluster.

    ```azurecli
    az account set --subscription "Subscription Name"  
    ```

4. Run the following command to identify the cluster location and resource ID:

    ```azurecli
    az openshift show -g <clusterResourceGroup> -n <clusterName>
    ```

5. Edit the JSON parameter file **existingClusterParam.json** and update the values *aroResourceId* and *aroResourceLocation*. The value for **workspaceResourceId** is the full resource ID of your Log Analytics workspace, which includes the workspace name.

6. To deploy with Azure CLI, run the following commands:

    ```azurecli
    az deployment group create --resource-group <ClusterResourceGroupName> --template-file ./ExistingClusterOnboarding.json --parameters @./existingClusterParam.json
    ```

    The output resembles the following:

    ```output
    provisioningState       : Succeeded
    ```

## Next steps

- With monitoring enabled to collect health and resource utilization of your RedHat OpenShift cluster and workloads running on them, learn [how to use](container-insights-analyze.md) Container insights.

- By default, the containerized agent collects the stdout/ stderr container logs of all the containers running in all the namespaces except kube-system. To configure container log collection specific to particular namespace or namespaces, review [Container Insights agent configuration](container-insights-agent-config.md) to configure desired data collection settings to your ConfigMap configurations file.

- To scrape and analyze Prometheus metrics from your cluster, review [Configure Prometheus metrics scraping](container-insights-prometheus-integration.md)

- To learn how to stop monitoring your cluster with Container insights, see [How to Stop Monitoring Your Azure Red Hat OpenShift cluster](./container-insights-optout-openshift-v3.md).