# Labb1 Documentation

---

## Step 1 Use Azure CLI in the terminal

```bash
az login
```

## Step 2 Look for my resource group name

```bash
az group list
```

Resource group name = YOUR-RESOURCE-GROUP-NAME

## Step 3 Create appservice plan

```bash
az appservice plan create \
--name MyAppServicePlan \
--resource-group YOUR-RESOURCE-GROUP-NAME \
--sku B1
```
default os would be window, or `--is-linux` to choose linux

## Step 4 Create Web App service

```bash
az webapp create \
--resource-group YOUR-RESOURCE-GROUP-NAME \
--plan MyAppServicePlan \
--name CommunityWebApi \
--runtime "DOTNET-VERSION"
```

## Step 5 Create Azure SQL Server

```bash
az sql server create \
--name CommunitySqlServer \
--resource-group YOUR-RESOURCE-GROUP-NAME \
--location swedencentral \
--admin-user YOUR-SQL-ADMIN-USER \
--admin-password YOUR-SQL-ADMIN-PASSWORD
```

## Step 6 Create database for the api project

```bash
az sql db create \
--resource-group YOUR-RESOURCE-GROUP-NAME \
--server CommunitySqlServer \
--name CommunityDB \
--service-objective Basic
```

## Step 7 Get the connection string

```bash
az sql db show-connection-string \
--name CommunityDB \
--server CommunitySqlServer \
--client ado.net
```
* Server = YOUR-SQL-SERVER
* User ID = YOUR-SQL-USER
* Password = YOUR-SQL-PASSWORD

## Step 8 Create Azure Key Vault and store the DB connection string in it

### 8.1 Create Azure Key Vault

```bash
az keyvault create \
--name YOUR-KEYVAULT-NAME \
--resource-group YOUR-RESOURCE-GROUP-NAME \
--location swedencentral
```

### 8.2 Find the user name and subscription id, which is need for the next step

```bash
az account show
```

### 8.3 Assign Key Vault Secrets Officer role to my Azure account

```bash
az role assignment create \
--assignee YOUR-USER-EMAIL \
--role "Key Vault Secrets Officer" \
--scope /subscriptions/YOUR-SUBSCRIPTION-ID/resourceGroups/YOUR-RESOURCE-GROUP-NAME/providers/Microsoft.KeyVault/vaults/YOUR-KEYVAULT-NAME
```

### 8.4 Store the connectionString in the key vault

```bash
az keyvault secret set \
--vault-name YOUR-KEYVAULT-NAME \
--name "ConnectionStrings--DefaultConnection" \
--value "YOUR-CONNECTION-STRING"
```

## Step 9 Get the APP service id and give the APP Service access

### 9.1 Get the object ID

```bash
az webapp identity assign \
--name CommunityWebApi \
--resource-group YOUR-RESOURCE-GROUP-NAME
```
look for `<principalId>: YOUR-PRINCIPAL-ID`

### 9.2 Assign a role to the app service

```bash
az role assignment create \
--assignee YOUR-PRINCIPAL-ID \
--role "Key Vault Secrets User" \
--scope /subscriptions/YOUR-SUBSCRIPTION-ID/resourceGroups/YOUR-RESOURCE-GROUP-NAME/providers/Microsoft.KeyVault/vaults/YOUR-KEYVAULT-NAME
```

### 9.3 Verify principalID

```bash
az webapp identity show \
--name CommunityWebApi \
--resource-group YOUR-RESOURCE-GROUP-NAME
```

## Step 10 Allow my App Service IPs

### 10.1 Get App Service outboundIpAddress and store them in variable IPS

```bash
IPS=$(az webapp show \
--name CommunityWebApi \
--resource-group YOUR-RESOURCE-GROUP-NAME \
--query outboundIpAddresses \
--output tsv)
```

### 10.2 Create firewall rule for all IPs

```bash
for ip in ${IPS//,/ } do
az sql server firewall-rule create \
--resource-group YOUR-RESOURCE-GROUP-NAME \
--server CommunitySqlServer \
--name AllowAppService-$ip \
--start-ip-address $ip \
--end-ip-address $ip
done
```

## Step 11 Enable https

```bash
az webapp update \
--name CommunityWebApi \
--resource-group YOUR-RESOURCE-GROUP-NAME \
--https-only true
```

## Step 12 Setup Azure DevOps
* Go to Azure DevOps and create a new project `CloudLabb1`
* In project settings under Pipeline add service connection
* Azure Resource Manager
* Service Connection Name `CloudLabb1Connection`
* Check grant access to all pipelines

## Step 13 Update the remote origin in the local project

```bash
git remote set-url origin YOUR-DEVOPS-REPOSITORY-URL
git push -u origin --all
```

## Step 14 Update local code to use Azure Key Vault

### 14.1 Install packages Azure.Identity

### 14.2 In appsettings.json

```json
KeyVault: {
  Url: "YOUR-KEYVAULT-URL"
}
```

### 14.3 In Program.cs

```csharp
var keyVaultUrl = builder.Configuration["KeyVault:Url"];

builder.Configuration.AddAzureKeyVault(
    new Uri(keyVaultUrl),
    new DefaultAzureCredential()
);
```

## Step 15 Create pipeline and use the yaml file example

```bash
# Vilken branch som triggar pipelinen eller none= manuell start
trigger:
- main

# Skapar en virtuell maskin med en windows image
# För self-hosted byt ut wmImage mot name: 'Namnet_DinAgentPool' 
pool:
  vmImage: windows-latest

# En variabel med värde som kan användas flera gånger
variables:
  buildConfiguration: 'Release'

# De olika stegen (steps och tasks) i skriptet (hämta kod-förbered-bygg-deploya)
steps:

# Hämta koden från samma repo(self) som YAML filen ligger i. För self-hosted lägg
# även till clean: true . Ny rad efter checkout. Det rensar tidigare körning
- checkout: self

# Installera .NET SDK och välj .NET 10 (x=alla versioner av den)
- task: UseDotNet@2
  inputs:
    packageType: 'sdk'
    version: '10.x'

# Restore av dependencies
- task: DotNetCoreCLI@2
  displayName: Restore
  inputs:
    command: 'restore'
    projects: '\*_/_.csproj'

# Gör build av projekt
- task: DotNetCoreCLI@2
  displayName: Build
  inputs:
    command: 'build'
    projects: '\*_/_.csproj'
    arguments: '--configuration $(buildConfiguration)'

# Publisera web app (skapa artifact i ArtifactStagingDirectory)
- task: DotNetCoreCLI@2
  displayName: Publish
  inputs:
    command: 'publish'
    publishWebProjects: true
    arguments: '--configuration $(buildConfiguration) --output $(Build.ArtifactStagingDirectory)'
    zipAfterPublish: true

# Deploya till Azure Web App (hämta från ArtifactStagingDirectory). För self
# hosted skall det se ut så här package: '$(Build.ArtifactStagingDirectory)
- task: AzureWebApp@1
  displayName: Deploy
  inputs:
    azureSubscription: 'CloudLabb1Connection'<YourProjectConnection>
    appType: 'webapp'
    appName: 'CommunityWebApi'
    package: '$(Build.ArtifactStagingDirectory)/\*_/_.zip'
```

## Step 16 Set up Environment Variable for the webapp service

```bash
az webapp config appsettings set \
--name CommunityWebApi \
--resource-group YOUR-RESOURCE-GROUP-NAME \
--settings \
ASPNETCORE_ENVIRONMENT="Development" \
Jwt:KEY="YOUR-JWT-KEY" \
Jwt:Issuer="YOUR-JWT-ISSUER" \
Jwt:Audience="YOUR-JWT-AUDIENCE" \
KeyVault:Url="YOUR-KEYVAULT-URL"
```

## Step 17 Enable Azure Insight

### 17.1 Create Monitor app-insights component

```bash
az monitor app-insights component create \
--app CommunityWebApi-AI \
--location swedencentral \
--resource-group YOUR-RESOURCE-GROUP-NAME \
--application-type web
```

### 17.2 Get the connectionString

```bash
az monitor app-insights component show \
--app CommunityWebApi-AI \
--resource-group YOUR-RESOURCE-GROUP-NAME
```

### 17.3 Add environment variable APPLICATIONINSIGHTS_CONNECTION_STRING

```bash
az webapp config appsettings set \
--name CommunityWebApi \
--resource-group YOUR-RESOURCE-GROUP-NAME \
--settings APPLICATIONINSIGHTS_CONNECTION_STRING="YOUR-CONNECTION-STRING"
```

### 17.4 Update local code to use Azure Insight

```csharp
builder.Services.AddApplicationInsightsTelemetry();
```

### 17.5 Go to Azure Insight, in the left-side bar, under Investigate, look for Live metrics

## Step 18 Test SQL server

### 18.1 Allow all IPs to access the server

```bash
az sql server firewall-rule create \
--resource-group YOUR-RESOURCE-GROUP-NAME \
--server CommunitySqlServer \
--name AllowAll \
--start-ip-address 0.0.0.0 \
--end-ip-address 255.255.255.255
```

### 18.2 Open SSMS
* Server Name = `YOUR-SQL-SERVER.database.windows.net`
* Authentication = `SQL Server Authentication`

## Step 19 Create Storage Account

```bash
az storage account create \
--name YOUR-STORAGE-ACCOUNT-NAME \
--resource-group YOUR-RESOURCE-GROUP-NAME \
--location swedencentral \
--sku Standard_LRS
```

## Step 20 Create Blob Container

### 20.1 Find key1

```bash
az storage account keys list \
--resource-group YOUR-RESOURCE-GROUP-NAME \
--account-name YOUR-STORAGE-ACCOUNT-NAME
```

### 20.2 create container

```bash
az storage container create \
--name uploads \
--account-name YOUR-STORAGE-ACCOUNT-NAME \
--account-key YOUR-STORAGE-ACCOUNT-KEY
```

### 20.3 Get the storage connection string

```bash
az storage account show-connection-string \
--name YOUR-STORAGE-ACCOUNT-NAME \
--resource-group YOUR-RESOURCE-GROUP-NAME
```

### 20.4 Store the connection string in key vault

```bash
az keyvault secret set \
--vault-name YOUR-KEYVAULT-NAME \
--name "Storage--ConnectionString" \
--value "YOUR-STORAGE-CONNECTION-STRING"
```

## Step 21 Enable daily backups for the web app

### 21.1 Create a new container for daily backups

```bash
az storage container create \
--name appservicebackups \
--account-name YOUR-STORAGE-ACCOUNT-NAME
```

### 21.2 Secure the container by adding a 30-day expiry date

```bash
EXPIRY=$(date -u -d "30 days" '+%Y-%m-%dT%H:%MZ')
```

### 21.3 Generate SAS token

```bash
az storage container generate-sas \
--account-name YOUR-STORAGE-ACCOUNT-NAME \
--name appservicebackups \
--permissions rwdl \
--expiry $EXPIRY \
--output tsv
```

### 21.4 Get the completed container URL
URL = `YOUR-BACKUP-CONTAINER-URL`

### 21.5 Configure the scheduled backup

```bash
az webapp config backup update \
--resource-group YOUR-RESOURCE-GROUP-NAME \
--webapp-name CommunityWebApi \
--container-url "YOUR-BACKUP-CONTAINER-URL" \
--frequency 1d \
--retention 7 \
--retain-one true
```

### 21.6 Verify the configuration

```bash
az webapp config backup show \
  --resource-group YOUR-RESOURCE-GROUP-NAME
```
should see :
```bash
`"backupSchedule": {
    "frequencyInterval": 1,
    "frequencyUnit": "Day",`
```
