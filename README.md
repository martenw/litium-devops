# Litium devops

Instructions on how to automatically build and deploy a Litium 8 Accelerator website to Litium Serverless Cloud.

> The old Litium 7 version can be found in the `litium-7` branch

Setup is based on:

- <https://docs.litium.com/platform/guides/deployment/azure-devops-and-webdeploy>
- <https://docs.litium.com/cloud/serverless/guides/automated-deployments>

This setup is prepared for two deployments, one to a dev-environment and one to a prod-environment.

## Add files to repo

Copy the `/pipelines`-folder from this directory to your repo root and push the change. Folder name is case-sensitive, if you change the name you must edit _workingDirectory_ in step _Authenticate, upload artifact and deploy_ to match.

## Setup in Azure devops

Do the steps below in your Azure devops project.

### Add cloud manifest to solution

To be able to identify the information of your release the `Litium.Cloud.Tools.Targets`-package need to be included in the application (for example in the `Litium.Accelerator.Mvc`-project)

[Read more](https://docs.litium.com/cloud/serverless/guides/artifacts/create-dotnet-artifact)

### Add service connection

Create a _Service Connection_ to access the Litium nuget feed.

1. In devops, select _Project Settings > Service connections_ and click on _New Service Connection_.
1. Select NuGet as type and click _Next_
1. Select _Basic Authentication_
1. Set **Feed URL** to `https://nuget.litium.com/nuget/` (the trailing slash is required!)
1. Set **Service Connection Name** to _Litium_ (name is used in YAML-build step later)
1. Set username and password to your Litium docs login
1. Select _Grant access permission to all pipelines_

Additional info can be found here: <https://docs.litium.com/platform/guides/deployment/azure-devops-and-webdeploy>

### Add Service Principal

A service principal is required to log in.

1. Create a service principal according to <https://docs.litium.com/cloud/serverless/guides/service-principal>:

   ```PowerShell
   litium-cloud service-principal create --file lcc.pem --name "devops-pipeline-litium" --expires 5000
   ```

   Check output of this command, it contains the e-mail assigned to the principal (in format _service.0123abc@cloud_), save this for the next step.

1. Grant [required permissions](https://docs.litium.com/cloud/serverless/guides/artifacts/deploy-artifact#access-control) to the new service principal

   ```PowerShell
   # subscription/reader
   litium-cloud subscription access-control add --email service.0123abc@cloud --role subscription/reader
   # environment/reader
   litium-cloud environment access-control add --email service.0123abc@cloud --role environment/reader --env prod
   # appresource/writer
   litium-cloud app access-control add --email service.0123abc@cloud --role appresource/writer --env prod --app litium
   # artifact/creator
   litium-cloud subscription access-control add --email service.0123abc@cloud --role artifact/creator
   ```

   _(run above for all environments)_

1. Upload the created `.pem`-file to _Devops > Pipelines > Library > Secure files_
1. Save the name of your `.pem`-file for later
1. Execute `litium-cloud subscription access-control show` to retreive

### Add Variable Groups

Go to _Devops > Pipelines > Library_ and add two variable groups with the values below, these values are referenced from the pipelines and should be named **LitiumTest** and **LitiumProd** (if any other name is used for variable groups you must also modify the references in the `.yml`-files).

The values you need here are provided by Litium for your application.

| Name                    | Value (replace \*)   | Info                                              |
| ----------------------- | -------------------- | ------------------------------------------------- |
| LitiumApp               | litium               |                                                   |
| LitiumCloudEnvironment  | \*\*\*               | For example _test_ or _prod_                      |
| LitiumCloudSubscription | \*\*\*               | For example _43qjdq2_                             |
| LitiumCloudUser         | service.\*\*\*@cloud | Mail of your service principal from earlier       |
| LitiumServicePrincipal  | lcc.pem              | Name of `.pem` file added to secure files earlier |

### Add pipelines

1. In _Devops > Pipelines_ click _New Pipeline_
1. Connect-step: Select _Azure Repos Git_
1. Select-step: Select your repo
1. Configure-step: Select _Existing Azure Pipelines YAML file_, go through this twice and add one pipeline for `azure-pipelines-test.yml` and one for `azure-pipelines-prod.yml`
