# ⚠️ This demo is no longer maintained
### ⚠️ Directions are kept for "how to" only. Please view [https://github.com/julie-ng/azure-nodejs-demo](https://github.com/julie-ng/azure-nodejs-demo) for an example Azure DevOps Pipeline.⚠️

---

### Azure Pipelines Example: Building and Deploying Containers

[![Build Status](https://dev.azure.com/julie-io/demos/_apis/build/status/julie-ng.azure-pipelines-acr-example?branchName=master)](https://dev.azure.com/julie-io/demos/_build/latest?definitionId=1&branchName=master)

This barebones repository shows how to automate building and pushing a Docker image and push to [Azure Container Registry](https://azure.microsoft.com/en-us/services/container-registry/) (ACR).

This was not straightfoward from the documentation. I got it working after trial and error and have published my simplified example. I hope you find it useful.

### Demo

This demo deploys an application to:

- **[https://azure-devops-acr-example.azurewebsites.net/health &rarr;](https://azure-devops-acr-example.azurewebsites.net/health)**

## Secure Registry Login with "Service Connection"

Instead of using `docker login`, we want to do a **secure login** by creating a [Service Connection using Azure Resource Manager](https://docs.microsoft.com/en-us/azure/devops/pipelines/library/connect-to-azure?view=azure-devops). This means we do not need a username or password in our environment.

```yaml
variables:
  azureSubscriptionEndpoint: <<YOUR_CONNECTION_NAME>>
  azureContainerRegistry: <<YOUR_ACR_NAME>>
  registryName: $(azureContainerRegistry)/$(Build.Repository.Name)
  imageTag: $(registryName):$(Build.BuildId)
```

You need to replace `<<VARIABLE>>` with your setup.

#### Example Values

| Variable | Example Value | Description |
|:--|:--|:--|
| `azureSubscriptionEndpoint` | `julie-ng-connection` | [Azure Service Connection Name](https://docs.microsoft.com/en-us/azure/devops/pipelines/library/connect-to-azure?view=azure-devops) |
| `azureContainerRegistry` | `julie.azurecr.io` | Your ACR Name, which should have the `azurecr.io` domain in it. |

## Docker Image Tags - append git shas

It is worth noting that the default Azure examples using just the build number as a tag, e.g. `42`. In my experience, a simple build number is not very helpful when debugging deployments. 

I prefer to append the git sha so that I easily know which code was deployed:

```
# Default Tag
julie-ng/azure-devops-acr-example:42

# Custom Tag with git sha
julie-ng/azure-devops-acr-example:69-07e19d5
```

If there's a problem in a deployment you can immediately view the image code with `git checkout -b <NEW_BRANCH_NAME> <GIT_SHA>`.


## Azure Pipeline Steps

After setting up the variables, the pipeline steps are mostly straightfoward - if you use the docker commands. I only kept the login task, because it has added security value. For the rest, plain docker commands are easier to understand IMO.

```yaml
steps:

- script: docker build --tag $(imageTag) .
  displayName: 'Docker: Build and tag image'

# Use the task to avoid embedding credentials in pipeline
- task: Docker@1
  displayName: 'Docker: login ACR'
  inputs:
    command: login
    azureSubscriptionEndpoint: $(azureSubscriptionEndpoint)
    azureContainerRegistry: $(azureContainerRegistry)

- script: docker push $(imageTag)-$(buildSha)
  displayName: 'Docker: Push image'

- script: docker logout $(azureContainerRegistry)
  displayName: 'Docker: logout ACR'

```

See full [azure-pipelines.yml](./azure-pipelines.yml) file for details.

## Deployment

Before you deploy, you need to create the resource (e.g. first deploy) by hand. Otherwise you may see this error message:

```
##[error]Error: Resource 'azure-devops-acr-example' doesn't exist. Resource should exist before deployment.
```

## References

- [Azure Pipelines: Build, test, and push Docker container apps](https://docs.microsoft.com/en-us/azure/devops/pipelines/languages/docker?view=azure-devops)
- [Azure Pipelines: Docker task](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/build/docker?view=azure-devops)
- [Azure Pipelines: Specify conditions](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/conditions?view=azure-devops&tabs=yaml)
- [Azure Pipelines > Deploy Tasks: Azure Web App for Container task](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/deploy/azure-rm-web-app-containers?view=azure-devops)
