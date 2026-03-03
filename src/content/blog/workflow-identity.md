---
title: 'Provision a Workflow Identity using Bicep'
description: |
  Uncovering the hidden, dramatic social order governed by the CSS z-index property.
  From the humble static div to the tyrannical modal overlay, every element knows its place.
pubDate: 'Mar 03 2026'
---

To save time during development, I like automating deployments using GitHub Actions. When I merge code into the main branch, a workflow provisions the required Azure resources. Then, the workflow deploys the app to those resources.

The workflow provisions Azure resources using the [`azure/bicep-deploy`](https://github.com/Azure/bicep-deploy) action. The recommended way for the workflow to connect to Azure is through [OIDC](https://github.com/azure/login?tab=readme-ov-file#login-with-openid-connect-oidc-recommended). To authorize the workflow through OIDC, I need to do four things:

- Create an identity for the workflow inside my Azure tenant.
- Assign the workflow identity the necessary permissions.
- Create a federated identity credential for the workflow identity.
- Upload the connection variables to the GitHub repository.

Doing these things by hand is tedious work. The [`azd pipeline config`](https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/configure-devops-pipeline) command streamlines this process. When the command runs, it first creates the identity for the workflow. Then, it uploads the variables of the connection to the GitHub repository of the project.

`azd pipeline config` is a great alternative to the manual setup. However, I prefer rolling my own workflow setup using a custom Bicep file. That way, I can version-control the provisioning of the workflow. I can also store the workflow identity in the same resource group as the other resources for the project.

## Microsoft Graph Permissions for Managed Identities

Sometimes, creating Azure resources isn't enough. The workflow may also need to set up Microsoft Graph resources. For some of my projects, I need to create app registrations, for example.

The [Microsoft Graph Bicep Extension](https://learn.microsoft.com/en-us/graph/templates/bicep/overview-bicep-templates-for-graph) lets me manage Graph resources using Bicep.

The Microsoft Graph app manages permissions related to Graph resources. In the Azure Portal, it is only possible to assign these permissions to app registrations. With Bicep, I can assign these permissions to other types of service principals.

## The Bicep File

Here is an annotated Bicep file I like to use to set up the workflow identity and its permissions.

This particular file assigns the [`Application.ReadWrite.OwnedBy`](https://learn.microsoft.com/en-us/graph/permissions-reference#applicationreadwriteownedby) Graph permission to the principal. This is an example of how to assign a Graph permission. The permission is not necessary to publish Azure resources.

The file also outputs the variables required by the GitHub repository.

```bicep
// /infra/workflow.bicep

// Import Microsoft Graph extension
extension 'br:mcr.microsoft.com/bicep/extensions/microsoftgraph/v1.0:1.0.0'

@description('E.g. markuslewin/weather-app')
param repo string

// Well-known ID of Microsoft Graph app
var graphAppId = '00000003-0000-0000-c000-000000000000'
// The [permissions reference](https://learn.microsoft.com/en-us/graph/permissions-reference) contains the IDs of possible permissions
var roleIdByName = {
  'Application.ReadWrite.OwnedBy': '18a4783c-866b-4cc7-a460-3d5e5662c884'
}

// The identity the workflow will use to provision resources
module identity 'br/public:avm/res/managed-identity/user-assigned-identity:0.4.1' = {
  params: {
    name: 'workflow'
    federatedIdentityCredentials: [
      {
        name: 'workflow-fic'
        issuer: 'https://token.actions.githubusercontent.com'
        subject: 'repo:${repo}:ref:refs/heads/main'
        audiences: [
          'api://AzureADTokenExchange'
        ]
      }
    ]
  }
}

// Authorize the workflow to create resources inside the resource group
module groupOwnerAssignment 'br/public:avm/res/authorization/role-assignment/rg-scope:0.1.0' = {
  params: {
    principalId: identity.outputs.principalId
    roleDefinitionIdOrName: 'Owner'
  }
}

// Find the service principal of the Microsoft Graph app
resource graphPrincipal 'Microsoft.Graph/servicePrincipals@v1.0' existing = {
  appId: graphAppId
}

// Authorize the workflow to create app registrations inside the tenant
resource appOwnerAssignment 'Microsoft.Graph/appRoleAssignedTo@v1.0' = {
  appRoleId: roleIdByName['Application.ReadWrite.OwnedBy']
  principalId: identity.outputs.principalId
  resourceId: graphPrincipal.id
}

// Secrets required by the `azure/login` and `azure/bicep-deploy` actions
output secrets {
  name: string
  secret: string
}[] = [
  {
    name: 'AZURE_TENANT_ID'
    secret: tenant().tenantId
  }
  {
    name: 'AZURE_SUBSCRIPTION_ID'
    secret: subscription().subscriptionId
  }
  {
    name: 'AZURE_RESOURCE_GROUP_NAME'
    secret: resourceGroup().name
  }
  {
    name: 'AZURE_CLIENT_ID'
    secret: identity.outputs.clientId
  }
]
```

## Deploying the Bicep File

The Bicep file deploys to a resource group, and so I need to create one before deploying the file:

```sh
# Create resource group
az group create --name my-rg --location eastus

# Provision workflow identity
az deployment group create \
  --resource-group my-rg \
  --template-file infra/workflow.bicep \
  --parameters "repo=markuslewin/weather-app"

# Cleanup
# az group delete --name my-rg
```

The `properties.outputs` field in the terminal output contains the required repository variables. I need to upload these variables to the repository that is going to run the workflow.

I automate this step by parsing the output of `az deployment`. Then, I upload the value to the repository using the [GitHub CLI](https://cli.github.com/).

Here is the final script. First, I create a resource group. Then, I add the workflow identity with the necessary permissions. Finally, I upload the secrets to the repository:

```sh
#!/bin/bash

export RESOURCE_GROUP="weather-app"
export LOCATION="eastus"
export REPO="markuslewin/weather-app"

# Create resource group
az group create --name "$RESOURCE_GROUP" --location "$LOCATION"

# Create federated identity for workflow
# Parse and set GitHub secrets for repo
gh secret set --repo "$REPO" --env-file <(
  az deployment group create \
    --resource-group "$RESOURCE_GROUP" \
    --template-file infra/workflow.bicep \
    --parameters "repo=$REPO" \
    --query properties.outputs.secrets.value \
      | jq -r '.[] | "\(.name)=\(.secret)"'
)

# This script creates all resources inside the resource group. To delete them:
# az group delete --name $RESOURCE_GROUP
```
