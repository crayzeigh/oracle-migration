# oracle-migration
Demo for oracle migration to OpenShift

References for building demo.

- [Medrec Sample App](https://github.com/vijaykumaryenne/medrec-sampleui-app)
- [Migrating Oracle to AzureSQL and PostgreSQL](https://github.com/Microsoft/MCW-Migrating-Oracle-to-Azure-SQL-and-PostgreSQL)

## Related References
- [Oracle to Azure Database for PostgreSQL Migration Cookbook](https://github.com/Microsoft/DataMigrationTeam/blob/master/Whitepapers/Oracle%20to%20Azure%20PostgreSQL%20Migration%20Cookbook.pdf)
- [Microsoft Cloud Workshops - Migrating Oracle to Azure SQL and PostgreSQL](https://github.com/microsoft/MCW-Migrating-Oracle-to-Azure-SQL-and-PostgreSQL/blob/master/Hands-on%20lab/Before%20the%20HOL%20-%20Migrating%20Oracle%20to%20Azure%20SQL%20and%20PostgreSQL.md)
Based on

## Target Audience
- Application Developer
- SQL Developer
- Database Administrator

## Azure services and related products
- OpenShift on AzureSQL
- Azure Database for PostgreSQL
- Azure Database Migration Service
- ora2pg
- Oracle

A Quickstart guide to migrating an Oracle Application to ARO.  This project provides working code for a demonstration using the MedRec application.

Working code for an Oracle Migration onto OpenShift.  This leverages [ora2pgsql](https://ora2pg.darold.net/)

Author: [Bryan Coapstick](https://twitter.com/bcoapstick), Aaron Aldrich

## Prerequisites

- Microsoft Azure subscription must be pay-as-you-go or MSDN
- Azure CLI [Installation Instructions](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli)

## ARO Quickstart

A Quickstart guide to deploying an Azure RedHat OpenShift cluster.  I copied it in for ease, but you can access the latest [here](https://mobb.ninja/docs/quickstart-aro.html) developed by [Paul Czarkowski](https://twitter.com/pczarkowski)

## Video Walkthrough

If you prefer a more visual medium, you can watch [Paul Czarkowski](https://twitter.com/pczarkowski) walk through this quickstart on [YouTube](https://youtu.be/VYfCltxoh40).

### Prepare Azure Account for Azure OpenShift

1. Log into the Azure CLI by running the following and then authorizing through your Web Browser

   ```bash
   az login
   ```

1. Make sure you have enough Quota (change the location if you're not using `East US`)

   ```bash
   az vm list-usage --location "East US" -o table
   ```

   see [Addendum - Adding Quota to ARO account](#adding-quota-to-aro-account) if you have less than `36` Quota left for `Total Regional vCPUs`.

1. Register resource providers

   ```bash
   az provider register -n Microsoft.RedHatOpenShift --wait
   az provider register -n Microsoft.Compute --wait
   az provider register -n Microsoft.Storage --wait
   az provider register -n Microsoft.Authorization --wait
   ```

### Get Red Hat pull secret

> This step is optional, but highly recommended

1. Log into cloud.redhat.com

1. Browse to https://cloud.redhat.com/openshift/install/azure/aro-provisioned

1. click the **Download pull secret** button and remember where you saved it, you'll reference it later.

## Deploy Azure OpenShift

### Variables and Resource Group

Set some environment variables to use later, and create an Azure Resource Group.

1. Set the following environment variables

   > Change the values to suit your environment, but these defaults should work.

   ```bash
   AZR_RESOURCE_LOCATION=eastus
   AZR_RESOURCE_GROUP=openshift
   AZR_CLUSTER=cluster
   AZR_PULL_SECRET=~/Downloads/pull-secret.txt
   ```

1. Create an Azure resource group

   ```bash
   az group create \
     --name $AZR_RESOURCE_GROUP \
     --location $AZR_RESOURCE_LOCATION
   ```


### Networking

Create a virtual network with two empty subnets

1. Create virtual network

   ```bash
   az network vnet create \
     --address-prefixes 10.0.0.0/22 \
     --name "$AZR_CLUSTER-aro-vnet-$AZR_RESOURCE_LOCATION" \
     --resource-group $AZR_RESOURCE_GROUP
   ```

1. Create control plane subnet

   ```bash
   az network vnet subnet create \
     --resource-group $AZR_RESOURCE_GROUP \
     --vnet-name "$AZR_CLUSTER-aro-vnet-$AZR_RESOURCE_LOCATION" \
     --name "$AZR_CLUSTER-aro-control-subnet-$AZR_RESOURCE_LOCATION" \
     --address-prefixes 10.0.0.0/23 \
     --service-endpoints Microsoft.ContainerRegistry
   ```

1. Create machine subnet

   ```bash
   az network vnet subnet create \
     --resource-group $AZR_RESOURCE_GROUP \
     --vnet-name "$AZR_CLUSTER-aro-vnet-$AZR_RESOURCE_LOCATION" \
     --name "$AZR_CLUSTER-aro-machine-subnet-$AZR_RESOURCE_LOCATION" \
     --address-prefixes 10.0.2.0/23 \
     --service-endpoints Microsoft.ContainerRegistry
   ```

1. Disable network policies on the control plane subnet

   > This is required for the service to be able to connect to and manage the cluster.

   ```bash
   az network vnet subnet update \
     --name "$AZR_CLUSTER-aro-control-subnet-$AZR_RESOURCE_LOCATION" \
     --resource-group $AZR_RESOURCE_GROUP \
     --vnet-name "$AZR_CLUSTER-aro-vnet-$AZR_RESOURCE_LOCATION" \
     --disable-private-link-service-network-policies true
   ```

1. Create the cluster

   > This will take between 30 and 45 minutes.

   ```bash
   az aro create \
     --resource-group $AZR_RESOURCE_GROUP \
     --name $AZR_CLUSTER \
     --vnet "$AZR_CLUSTER-aro-vnet-$AZR_RESOURCE_LOCATION" \
     --master-subnet "$AZR_CLUSTER-aro-control-subnet-$AZR_RESOURCE_LOCATION" \
     --worker-subnet "$AZR_CLUSTER-aro-machine-subnet-$AZR_RESOURCE_LOCATION" \
     --pull-secret @$AZR_PULL_SECRET
   ```

1. Get OpenShift console URL

   ```bash
   az aro show \
     --name $AZR_CLUSTER \
     --resource-group $AZR_RESOURCE_GROUP \
     -o tsv --query consoleProfile
   ```

1. Get OpenShift credentials

   ```bash
   az aro list-credentials \
     --name $AZR_CLUSTER \
     --resource-group $AZR_RESOURCE_GROUP \
     -o tsv
   ```

1. Use the URL and the credentials provided by the output of the last two commands to log into OpenShift via a web browser.

![ARO login page](./images/aro-login.png)

1. Deploy an application to OpenShift

   > See the following video for a guide on easy application deployment on OpenShift.

### Delete Cluster

Once you're done its a good idea to delete the cluster to ensure that you don't get a surprise bill.

1. Delete the cluster

   ```bash
   az aro delete -y \
     --resource-group $AZR_RESOURCE_GROUP \
     --name $AZR_CLUSTER
   ```

1. Delete the Azure resource group

   > Only do this if there's nothing else in the resource group.

   ```bash
   az group delete -y \
     --name $AZR_RESOURCE_GROUP
   ```

## Addendum

### Adding Quota to ARO account

![aro quota support ticket request example](./images/aro-quota.png)

1. [Create an Azure Support Request](https://portal.azure.com/#blade/Microsoft_Azure_Support/HelpAndSupportBlade/newsupportrequest)

1. Set **Issue Type** to "Service and subscription limits (quotas)"

1. Set **Quota Type** to "Compute-VM (cores-vCPUs) subscription limit increases"

1. Click **Next Solutions >>**

1. Click **Enter details**

1. Set **Deployment Model** to "Resource Manager

1. Set **Locations** to "(US) East US"

1. Set **Types** to "Standard"

1. Under **Standard** check "DSv3" and "DSv4"

1. Set **New vCPU Limit** for each (example "60")

1. Click **Save and continue**

1. Click **Review + create >>**

1. Wait until quota is increased.
