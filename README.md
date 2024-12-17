# Litium devops

Instructions on how to automatically build and deploy a Litium 8 Accelerator website to Litium Serverless Cloud.

> The old Litium 7 version can be found in the `litium-7` branch

Setup is based on:

- <https://docs.litium.com/platform/guides/deployment/azure-devops-and-webdeploy>
- <https://docs.litium.com/cloud/serverless/guides/automated-deployments>

This setup is prepared for two deployments, one to a dev-environment and one to a prod-environment.

## Add files to repo

Add the `/pipelines`-folder to your repo root and push the change.

## Setup in Azure devops

Do the steps below in your Azure devops project.

### Add service connection

Create a _Service Connection_ to access the Litium nuget feed.

1. In devops, select _Project Settings > Service connections_ and click on _New Service Connection_.
1. Set **Connection name** to _Litium_ (name is used in YAML-build step later)
1. Set **Feed URL** to `https://nuget.litium.com/nuget`
1. Set username and password to your Litium docs login

Additional info can be found here: <https://docs.litium.com/platform/guides/deployment/azure-devops-and-webdeploy>

### Add Service Principal

A service principal is required to log in.

1. Create a service principal according to <https://docs.litium.com/cloud/serverless/guides/service-principal>
1. Upload the created `.pem`-file to _Devops > Pipelines > Library > Secure files_
1. Save the name of your `.pem`-file for later

### Add Variable Groups

Add 2 variable groups with the values below, these values are referenced from the pipelines and should be named **LitiumDev** and **LitiumProd** (if any other name is used for variable groups you must also modify the references in the `.yml`-files).

- The values you need here are provided by Litium for your application.
- Add in _Devops > Pipelines > Library > Variable groups_

| Name                    | Value (replace \*)   | Info                                              |
| ----------------------- | -------------------- | ------------------------------------------------- |
| LitiumApp               | litium               |                                                   |
| LitiumCloudEnvironment  | \*\*\*               | For example _dev_ or _prod_                       |
| LitiumCloudSubscription | \*\*\*               | For example _43qjdq2_                             |
| LitiumCloudUser         | service.\*\*\*@cloud |                                                   |
| LitiumServicePrincipal  | \*\*\*.pem           | Name of `.pem` file added to secure files earlier |

### Add pipelines

1. In _Devops > Pipelines_ click _New Pipeline_
1. Connect-step: Select _Azure Repos Git_
1. Select-step: Select your repo
1. Configure-step: Select _Existing Azure Pipelines YAML file_, go through this twice and add one pipeline for `azure-pipelines-dev.yml` and one for `azure-pipelines-prod.yml`
