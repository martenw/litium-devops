# Litium devops

Instructions on how to automatically build and deploy a Litium Accelerator website to Litium Cloud using Microsoft Devops Server.

## Preparations

Create a [Service Connection](https://docs.microsoft.com/en-us/azure/devops/pipelines/library/service-endpoints) to the Litium nuget feed. Credentials defined in the Service Connection is used to connect to Litium Nuget.

1. In devops, select *Project Settings > Service connections*
1. Set **Connection name** to *LitiumNuget* (name is used in YAML-build step later)
1. Set **Feed URL** to *https://nuget.litium.com/nuget*
1. Set username and password to  your Litium docs login

## Build

First add the NuGet.config file to your checked in source code in devops and reference the file in the `NuGetCommand@2` task in YAML below.

Then add the build:

1. In devops select *Pipelines > Builds > New > New build pipeline*
1. Select Azure repos git (YAML)
1. Select your repository
1. In the next step Configure your pipeline select ASP.NET
1. Adjust `pool` in the YAML script below and use it to replace the generated YAML in devops
1. Run the build, this publishes the website to the *LitiumBuildArtifact*

YAML build script:
```YAML
# ASP.NET
# Build and test ASP.NET projects.
# Add steps that publish symbols, save build artifacts, deploy, and more:
# https://docs.microsoft.com/azure/devops/pipelines/apps/aspnet/build-aspnet-4
# https://docs.microsoft.com/en-us/azure/devops/pipelines/yaml-schema

# Automatically trigger build when code is pushed to master or develop branches
trigger:
- master
- develop

# Which pool to use for the pipeline
# https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/pools-queues
pool:
  name: 'RND'

variables:
  solution: '**/*.sln'
  buildPlatform: 'Any CPU'
  buildConfiguration: 'Release'

steps:
- task: NuGetToolInstaller@0

- task: NuGetCommand@2
  inputs:
    restoreSolution: '$(solution)'
    feedsToUse: config
    nugetConfigPath: 'Resources/NuGet.config'
    externalFeedCredentials: 'LitiumNuget'

- script: yarn install
  workingDirectory: 'Src/Litium.Accelerator.Mvc'

- script: yarn run prod
  workingDirectory: 'Src/Litium.Accelerator.Mvc'

- task: VSBuild@1
  inputs:
    solution: '$(solution)'
    msbuildArgs: '/p:Configuration=Release /p:DeployOnBuild=true /p:DeployDefaultTarget=WebPublish /p:WebPublishMethod=FileSystem /p:SkipInvalidConfigurations=true /p:DeleteExistingFiles=True /p:publishUrl="$(build.artifactStagingDirectory)"'
    platform: '$(buildPlatform)'
    configuration: '$(buildConfiguration)'

# TODO run tests, requires DB
# - task: VSTest@2
#   inputs:
#     platform: '$(buildPlatform)'
#     configuration: '$(buildConfiguration)'

- task: PublishBuildArtifacts@1
  inputs:
    pathToPublish: $(Build.ArtifactStagingDirectory)
    artifactName: LitiumBuildArtifact
```

## Deploy

1. In devops select *Pipelines > Releases > New > New release pipeline*
1. Select to start with an *Empty job*
1. For artifact select the *LitiumBuildArtifact* created in the build
1. Optionally click the lightning-symbol on artifacts to also enable automated deploy on build

### Add deploy tasks:

First a *Config transformation*-task to set the proper database connectionstring:
    
1. Set path to: `$(System.DefaultWorkingDirectory)/_Litium.Workshop.BestPractice/LitiumBuildArtifact/Web.config`
1. Set **File type** to *Auto detect*
1. Set **Target** to *source file*
1. Set **Type** to *Inline JSON*
1. Adjust the snippet below to your connectionstring and copy the value to the **Transformations**-field:
   ```
   {
       "configuration/connectionStrings/add[@name='FoundationConnectionString']/@connectionString": "Pooling=true;Database=TODO;Server=TODO;Integrated security=true;MultipleActiveResultSets=True"
    }
   ```

Next add a *SFTP Upload* task (the task can be found on [marketplace](https://marketplace.visualstudio.com/items?itemName=jean-marc-ducasse.sftpupload)):

1. As **Source folder** select the *LitiumBuildArtifact*
1. As **Contents** set **
1. As **Target folder** set */customer-name/wwwroot* - IMPORTANT - make sure you get this right to not overwrite another customers data!
1. As host name set the Litium SFTP-server, server 2 is required for the task to work
1. As username set your LITIUMDRIFT-account name
1. As Password set `$(sftppassword)`

Next set up the `sftppassword`-parameter that is used in the SFTP-task:

1. Select *Library* and create a new *Variable group*
1. Create a variable in the new variable group called `sftppassword`, as value set the password of your LITIUMDRIFT-account
1. Click the lock so that the new variable is not readable
1. Next edit the *Release* created earlier
   1. Select the Variables-tab of the release
   1. Select variable groups and click *Link variable group* where you select the new variable group

