# Example Deployment Steps

> **_NOTE:_** For our reference implementation, we are providing a mechanism that you can use to deploy this workload to your own subscription. These steps are not part of the reference implementation, but does represent the type of work that needs to be done. You would typically encapsulate this work in your continuous delivery pipeline (Azure DevOps, Jenkins, etc) in a way that aligns with your operational practices.

## Execute the Provided Deployment Script via Azure Cloud Shell

The provided `rundeployment.sh` will create all dependencies necessary for the web application and will deploy the web application infrastructure as well. Open an Azure Cloud Shell and run below commands, replacing the \[yourValues\] as appropriate.

```bash
mkdir deployweb
cd deployweb

export DEPLOYMENT=https://raw.githubusercontent.com/mspnp/reference-architectures/master/web-app/deployment/
export RGNAME=[yourResourceGroupName]
export RGLOCATION=[yourAzureRegionIdentifier]
export SQLSERVERNAME=[yourSqlServerName]
export SQLSERVERDB=[youSqlServerDb]
export SQLADMINUSER=[yourSqlAdminUser]
export DNSNAME=[yourGloballyUniqueDNSNameOfWebApp]
export STORAGEACCNAME=[yourGloballyUniqueStorageAccountName]
```

> **_NOTE:_**  The DNS Name of your web application needs to be globally unique. You can use the following command to validate that the name is unique before you run the rest of the script. If you find the name is already used, change `DNSNAME` to another value.

```bash
subscription=`az account show -o tsv --query id`
token=`az account get-access-token -o tsv --query accessToken`

curl -H "Authorization: Bearer ${token}" "https://management.azure.com/subscriptions/${subscription}/providers/Microsoft.Network/locations/${RGLOCATION}/CheckDnsNameAvailability?domainNameLabel=${DNSNAME}&api-version=2018-11-01"
```

> **_NOTE:_**  The Storage Account Name of your web application needs to be globally unique. You can use the following command to validate that the name is unique before you run the rest of the script. If you find the name is already used, change `STORAGEACCNAME` to another value.

```bash
az storage account check-name -n ${STORAGEACCNAME}
```

SQL Database accounts (including the admin) have a minimum password size of eight characters ([amongst other requirements](https://docs.microsoft.com/sql/relational-databases/security/password-policy?view=azuresqldb-current)). Capture a suitable password into `SQLADMINPASSWORD`.

```bash
read -s SQLADMINPASSWORD
export SQLADMINPASSWORD
```

Download `rundeployment.sh` from this repo and execute it. This will take about 20 minutes to execute.

```bash
wget ${DEPLOYMENT}rundeployment.sh
chmod +x rundeployment.sh
./rundeployment.sh
```

At this point you have all of the Azure resources in place: SQL Database, Cosmos DB, App Service, Application Insights, Azure Cache for Redis, Azure Front Door, Azure Service Bus, and Azure Storage.  There is no content in Cosmos DB nor is the web application code itself yet deployed.

## Populate Cosmos DB Starter Content (Optional)

The Cosmos DB server you deployed has a container named `cacheContainer` that is designed to hold advertisements for the website's footer. While they are not required for the Reference Implementation to function here is an example of content you could include. The script you ran above dropped a **Microsoft_Azure_logo_small.png** file into the storage account. We can reference that file in a fake ad.

```bash
imageUrl=`az storage account show -n ${STORAGEACCNAME} | jq -r .primaryEndpoints.blob`rsrcontainer/Microsoft_Azure_logo_small.png
echo $imageUrl
```

```json
{"id": "1","Message": "Powered by Azure","MessageType": "AD","Url": "[yourImageUrlHere]"}
```

Using the Azure Portal, Azure CLI, or Azure Storage Explorer add this document to the `cacheContainer` container in the Cosmos DB Server created above.

To do this from the Azure Portal, in the resource group of deployment, click on **Azure Cosmos Db Account** then select **cacheContainer** then click on **Documents**. Click on **New Document**. Replace the whole json payload with above content and click **Save**.

## Publish Web Application and Azure Function

We'll publish the web applications directly from Visual Studio. As with the resources above, this would normally be performed via your continuous delivery pipeline in Azure DevOps, Jenkins, etc.

1. Clone the repo.
1. Open **Voting.sln** solution.
   1. **Deploy the Voting API**
      1. Right click on the **VotingData** project. Click on **Publish**, select **new profile** then click on **existing**. Select the resource group for the deployment. Select the VotingData api app service deployment.
   1. **Deploy the Voting website**
      1. Right click on the **VotingWeb** project. Click on **Publish**, select **new profile** then click on **existing**. Select the resource group for the deployment. Select the VotingWeb app service deployment.
1. **Deploy the Vote Counter Function App**
   1. Open **FunctionVoteCounter.sln** solution
   1. Right click on **VoteCounter** project. Click on **Publish**, select **new profile** then click on **existing**. Select the same resource group as above. Select the function app service deployment.

Your website is fully deployed now. You can open the url <https://yourwebfrontend.yourlocation.cloudapp.azure.com/>, ignoring the certificate validation error on the browser.
