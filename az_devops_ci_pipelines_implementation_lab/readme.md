Clone the repo here https://github.com/dockersamples/example-voting-app.git and the follow the instructions here https://github.com/dockersamples/example-voting-app?tab=readme-ov-file tu run it locally

- the voting app is written in python
- redis as in-memory data storage
- the worker is written in .net. It takes the information from redis and stores it in a postgres database
- the result microservice is reading from the db

First we create a new project create. On the azure portal under `services` i look for `My Azure DevOps Organizations`.
On the Azure DevOps, select `new project` in order to create a new one.
![alt text](https://github.com/kokouvishna/az_devops_cicd_pipelines__lab/tree/main/az_devops_ci_pipelines_implementation_lab/images/new_prjct.PNG)

Now we need to import the business app into the new created project. We head to the app github page and copy the HTTPS url. Now we head back to our project on azure devops -> Repos -> Import a repository; and paste the link in the `Clone URL *` field, then import.
![alt text](az_devops_ci_pipelines_implementation_lab/images/import_repo.PNG)
After a while the import will be successful.

AZ devops selects alphabetically a default branch, make sure to switch to main. Under `Branches`, star `main` or use the 3 dots to set it as default branch.
![alt text](az_devops_ci_pipelines_implementation_lab/images/default_brnch.PNG)

First let's create a resource group
Let's head to the azure portal > under `Azure Services` select `Resource Groups` > `create`
Let's call our resource group `azurecicd`, then `Review + create` > `create`
![alt text](az_devops_ci_pipelines_implementation_lab/images/create_rg.PNG)

Let's head to the azure portal, look for `Container registries`
![alt text](az_devops_ci_pipelines_implementation_lab/images/container_registry_search.PNG)

Now let's create a new container registry called `labazurecicd`
![alt text](az_devops_ci_pipelines_implementation_lab/images/create_container_reg.PNG)
`Review + create` > `create`
After a while the registry will be created.

Now let's create the pipelines. We need to create 3, because we have 3 microservices.
Let's head to the azure devops page > `voting-app` > `Pipelines` > `Create Pipeline`. Under `Connect` we select `Azure Repos Git`
![alt text](az_devops_ci_pipelines_implementation_lab/images/select_az_repo_git.PNG)
Under `Select` choose `voting-app`.
Under `Configure` > `Docker` (build and push an image to azure container registry) > choose your azure subscription > `Continue`. 
A new window will pop up, log in with your azure portal credentials.
After logging in, go back to the main webbrowser window, under the field `Container registry` select the new created registry `labazurecicd` > `Validate and configure`.
![alt text](az_devops_ci_pipelines_implementation_lab/images/create_1st_pl.PNG)
As you can see, in the Dockerfile field there is `$(Build.SourcesDirectory)/result/Dockerfile`. The keyword `result` means, this pipeline is dedicated to the `result` microservice.
Now under `Review` there is a new file created named `azure-pipelines.yml`

`azure-pipelines.yml`
```yaml
# Docker
# Build and push an image to Azure Container Registry
# https://docs.microsoft.com/azure/devops/pipelines/languages/docker

trigger:
- main

resources:
- repo: self

variables:
  # Container registry service connection established during pipeline creation
  dockerRegistryServiceConnection: '29c997cb-d69d-4571-864c-64adff5243e8'
  imageRepository: 'votingapp'
  containerRegistry: 'labazurecicd.azurecr.io'
  dockerfilePath: '$(Build.SourcesDirectory)/result/Dockerfile'
  tag: '$(Build.BuildId)'

  # Agent VM image name
  vmImageName: 'ubuntu-latest'

stages:
- stage: Build
  displayName: Build and push stage
  jobs:
  - job: Build
    displayName: Build
    pool:
      vmImage: $(vmImageName)
    steps:
    - task: Docker@2
      displayName: Build and push an image to container registry
      inputs:
        command: buildAndPush
        repository: $(imageRepository)
        dockerfile: $(dockerfilePath)
        containerRegistry: $(dockerRegistryServiceConnection)
        tags: |
          $(tag)
```
The yaml file is structured as follow: `trigger`, `resources`, `variables` and `stages`.
- Trigger: tells the azure pipeline when the pipeline should trigger automatically. For example if someone makes a change in one of the 3 microservices, its pipeline should be automatically triggered. To implement this trigger we will be using a `path based trigger`.
- Stages: we will be running jobs for example for building, pushing and testing stages.
- Variables: variables will be using

Now let's rename `azure-pipelines.yml` to `azure-pipelines-result.yml` make changes to the file:
- under `trigger`, we're now using a path based trigger pointing to the result folder
- we removed the `Agent VM image name` and created our own with `pool`, an agent ontop of which the pipelines will run.

- under `stages`, click on `settings` to perform some commands. As first job in the build stage, we just want to build, so we select `build` under `Commands` > `Command*`. Under `Commands` > `Dockerfile*` we specify where the Dockerfile of the `result` microservice can be found. We finish the settings with `add`.
After the build stage, now we add another one, the push stage. We click again on this stage settings, but this time because we want to perform a push, under `Commands` > `Command` we select `push` > `add`.
Our result pipeline is now done, and we select `Save and run`.
![alt text](az_devops_ci_pipelines_implementation_lab/images/result_pl_unsaved.PNG)

`azure-pipelines-result.yml`
```yaml
# Docker
# Build and push an image to Azure Container Registry
# https://docs.microsoft.com/azure/devops/pipelines/languages/docker

trigger:
 # path based trigger
 paths:
   include:
     # everything under the result folder will trigger automatically
     - result/*

resources:
- repo: self

variables:
  # Container registry service connection established during pipeline creation
  dockerRegistryServiceConnection: '29c997cb-d69d-4571-864c-64adff5243e8'
  # image repository within the container registry
  imageRepository: 'resultapp'
  containerRegistry: 'labazurecicd.azurecr.io'
  dockerfilePath: '$(Build.SourcesDirectory)/result/Dockerfile'
  tag: '$(Build.BuildId)'

# Agent VM image name: azure agent on which all pipelines will run
pool:
 name: 'azureagent'

stages:
- stage: Build
  displayName: Build
  jobs:
  - job: Build
    displayName: Build
    steps:
    - task: Docker@2
      displayName: Build an image to container registry
      inputs:
        containerRegistry: '$(dockerRegistryServiceConnection)'
        repository: '$(imageRepository)'
        command: 'build'
        Dockerfile: 'result/Dockerfile'
        tags: '$(tag)'
- stage: Push
  displayName: Push
  jobs:
  - job: Push
    displayName: Push
    steps:
    - task: Docker@2
      displayName: Push an image to container registry
      inputs:
        containerRegistry: '$(dockerRegistryServiceConnection)'
        repository: '$(imageRepository)'
        command: 'push'
        tags: '$(tag)'

```
Again `Save and run` to save and run
![alt text](az_devops_ci_pipelines_implementation_lab/images/result_pl_save_n_run.PNG)

The jobs should not run as expected bevause we encounter an error, due to non-existence of an agent pool. Although we defined one in the yaml we didn't create it. 
![alt text](az_devops_ci_pipelines_implementation_lab/images/ci_no_agent_pool_error.PNG)
Now let's create our agent. Let's head to the azure portal and search for `Virtual machines` > `create` > `Azure virtual machine`:
- `azurecicd` as resource group
- `azureagent` as virtual machine name
- `Ubuntu Server 20.04 LTS - x64 Gen2` as image
- `Standard_B1s` as size
- `SSH public key` as authentication type
- `Review + create` > `create`
![alt text](az_devops_ci_pipelines_implementation_lab/images/agent_pool_vm_create.PNG)
Also make sure to download the your *.pem file and store it somewhere safe.
Now that the agent pool is created, go to the resource > `Overview` and copy the ip address. Now let's integrate this virtual machine to the azure devops platform. This azure devops services documentation explains clearly the steps that should be done in order to integrate the virtual machine agent pool into the devops platform https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/linux-agent?view=azure-devops
![alt text](az_devops_ci_pipelines_implementation_lab/images/setting_up_agent_pool.PNG)
After setting up our agent pool named `azureagent`, click on it > `New agent` > `Linux`. Follow the instructions, copy and execute the commands in the agent pool virtual machine.
![alt text](az_devops_ci_pipelines_implementation_lab/images/agent_configuration.PNG)
If you are on linux or macos just open your console, but on windows download and install `git bash`.
Go to the directory where the .pem file is saved, open a console and run following:
```shell
ssh -i azure_user_key_pair.pem azureuser@<vm-ip-address>
```
confirm then with yes. Now you are logged into your agent pool virtual machine and it should look like this:
```shell
azureuser@azureagent:~$
```
Run following comman to update the system because it's new and then install docker:
```shell
sudo apt update
sudo apt install -y docker.io
```
Once docker is up and running we grant the azuer user permission to the docker deamon.
```shell
sudo usermod -aG docker azureuser
sudo systemctl restart docker
```

Run following commands in order to download, create an configure the agent.
- create a folder for the agent:
```shell
mkdir myagent && cd myagent
```
- download the agent. First copy the download link from this page:
![alt text](az_devops_ci_pipelines_implementation_lab/images/agent_configuration.PNG)
```shell
wget https://vstsagentpackage.azureedge.net/agent/4.248.0/vsts-agent-linux-x64-4.248.0.tar.gz
```
- configure the agent:
```shell
./config.sh
```
Server URL: https://dev.azure.com/{your-organization}
Your organization is not the project name, but the name shown on your azure devops platform.
Your also also a personal access tocken (PAT). Go to azure devops > `User settings`  > `Personal Access Tokens` > `New Token` > `Create`
![alt text](az_devops_ci_pipelines_implementation_lab/images/create_pat.PNG)
Also make sure to save the PAT somewhere safe.
Now that we have the PAT we can go back the console where we logged into the vm, and enter the PAT when prompted. When also prompted to enter the agent pool, enter the name of the one we created earlier named `azureagent`.
When prompted again only confirm with enter.
Now that the agent pool is configured, to run it execute:
```shell
./run.sh
```
![alt text](az_devops_ci_pipelines_implementation_lab/images/agent_pool_console_config.PNG)
To see if our agent is really running and active, head to `azure devops` > `Settings` > `Agent pools` > `azureagent` > `Agents`. Our agent should be `Online`.
![alt text](az_devops_ci_pipelines_implementation_lab/images/check_agent_pool_status.PNG)

Now that we have a running agent, if we head back to azure devops > ``voting-app`` > ``Pipelines`` > ``voting-app``, and run again the failed pipeline, it should now pass.
![alt text](az_devops_ci_pipelines_implementation_lab/images/1st_result_job_passed.PNG)

In the agent pool console we should also have this output
![alt text](az_devops_ci_pipelines_implementation_lab/images/proof_1st_result_job.PNG)


We can to create a docker image, then push it to the azure container registry
