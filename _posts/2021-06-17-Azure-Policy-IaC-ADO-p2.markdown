---
layout: post
title:  "Managing Azure Policies with Infrastructure as Code principles: configuring diagnostic logs (Part 2)"
date:   2021-06-17
description: In this post we’ll talk about one of the most important part of Azure Governance – Azure Policy. I will describe how to manage policies using Infrastructure as Code (IaC) principles and pipelines in Azure DevOps.
categories:
  - Azure
tags:
  - Azure
  - Azure Governance
  - Azure Policy
  - Log Analytics
---

{:refdef: style="text-align: center;"}
![Title image]({{ site.baseimg }}\assets\img\2021\2021-06-17\title.jpg)
{: refdef}


<p class="intro"><span class="dropcap">I</span>n this post we’ll talk about one of the most important part of Azure Governance – Azure Policy. In this part there will be overview of Azure policies together with explaining how to use it for configuring Diagnostic Logs for Azure resources</p>

### Managing policies as Infrastructure as Code in Azure DevOps

Infrastructure as Code is a key principle of DevOps practice. This principle allows to describe cloud infrastructure via approved code and manage it through automatic provisioning using pipelines in from DevOps tools. One of the most important benefits of Iac is to create repetitive deployments and avoid human errors. This is primarily used to deploy resources and services, but government components like policies could be deployed as IaC too.

![Policies as code workflow](\assets\img\2021\2021-06-17\policy-as-code-workflow.png)

There are several way how to achieve this. Microsoft describes general concepts in [Design Azure Policy as Code workflows article](https://docs.microsoft.com/en-us/azure/governance/policy/concepts/policy-as-code) and offers [easy integration of policies as code with GitHub](https://docs.microsoft.com/en-us/azure/governance/policy/tutorials/policy-as-code-github). Policies could also be described as Azure blueprint (another part of Azure Governance) artifacts. However, Azure doesn't provide out of the box automation for blueprints, so IaC principles could be applied to them too.

My solution is based on [Tao Yang's brilliant blog series](https://blog.tyang.org/2019/09/08/configuring-azure-management-group-hierarchy-using-azure-devops/) but with several major changes. Management group structure is defined in code repository in different way and all scripts are written by me. 

Here are some highlights of this solution:

- Solution consists of three components: repository describing policies and management group, build pipeline performing tests of definition and assignment files in JSON format and release pipeline publishing it all to Azure
- Solution automates  deployment and assignment for Azure initiatives (aka policy set)
- Automation is performed for a single tenant. It's possible to extend it for multi-tenant scenarios adding more root management group to root repo directory and creating additional build adn release pipelines with separate ADO service connection per each tenant
- Build pipeline is enabled for continuous integration and triggers for every commit
- **Important: make sure microsoft.insights provider is registered for a subscription(s) before assigning policy configuring Diagnostic Logs otherwise the following error might occur**

![microsoft.insights provider is not registered](\assets\img\2021\2021-06-17\Diag-Setting-Error.png)

#### Code repository

Directory structure for policy management reflects hierarchy of management group in a tenant. Tenant root management group resides in root repo directory and has name corresponding to MG GUID. Child groups are sub-directories of this folder. Policy initiatives are created in directory `_policy`. Every policy set is located inside *policy.json* file (hardcoded in scripts). Policy set assignments always have JSON file names prefixed with 'assignment.'. This structure is similar to one described in [Microsoft article about policy as code in GitHub](https://docs.microsoft.com/en-us/azure/governance/policy/tutorials/policy-as-code-github).

![management group structure](\assets\img\2021\2021-06-17\ADO-repo-structure.png)

#### Service connection in Azure DevOps

Service connection in ADO is a way to connect to services to perform various task. When talking about Azure, technically it usually either [unmanaged or managed (managed identity linked to VM) service principle](https://docs.microsoft.com/en-us/azure/devops/pipelines/library/connect-to-azure?view=azure-devops). It could be created manually or from ADO portal. Service principle created from the portal provides Owner role to the chosen level.

![ADO Service principle](\assets\img\2021\2021-06-17\ADO-Service-Connector.png)

However we might want to assign more granular permissions for pipelines deploying and assigning policies. For our pipeline we need following permissions:
1. Create deployment. Permission:
* `Microsoft.Resources/deployments/*`
2. Deploy and assign policies. Permissions: 
* `Microsoft.Authorization/policyassignments/*`
* `Microsoft.Authorization/policyDefinitions/*`
* `Microsoft.Authorization/policysetdefinitions/*`
3. Assign IAM roles for managed identities of `DeployIfNotExists` policies. Permission: 
* `Microsoft.Authorization/roleAssignments/*`

Therefore if we want one role, we'd need to create custom one. For simplicity I'll use Owner role for this example. This indeed makes protection access ADO project where service connector is stored important the same way as access to Azure tenant itself.


#### Build pipeline

Build pipeline (shown as 'Pipeline' in ADO GUI) is normally used to compile and build software code. When performing Infrastructure as Code there is no need to build template files. We'll perform test JSON files before trying to publish and assigning them. 

I simplify the build pipeline and don't perform complex tests. Test is done using Pester and all it does is checking if JSON files have correct format. [Tao Yang developed tests specific to Azure policies](https://blog.tyang.org/2019/05/19/deploying-azure-policy-definitions-via-azure-devops-part-2/) that could be used instead.

Pipeline is [defined using classic Azure DevOps interface](https://docs.microsoft.com/en-us/azure/devops/pipelines/get-started/pipelines-get-started?view=azure-devops#define-pipelines-using-the-classic-interface) and consists of four tasks:

1. powershell performing tests
2. Publish test results from 1st task in XML format
3. Copy scripts and whole MG structure from repo for a build pipeline
4. Publishing artifacts for a build pipeline

![Build pipeline](\assets\img\2021\2021-06-17\build-pipeline.png)

The pipeline is configured for CI. To make it multi-tenant, specify root management group all tenants per each build pipeline.   

![pipeline continuous integration](\assets\img\2021\2021-06-17\build-pipeline-CI.png)

```yaml
pool:
  name: Azure Pipelines
steps:
- powershell: |
   import-module pester -Force -RequiredVersion 3.4.0
   Import-Module $(Build.SourcesDirectory)\_scripts\Tests\Test-JSON-Content.psm1
   Test-JSONContent -Path "$(Build.SourcesDirectory)" -OutputFile "TEST--JSON-blueprint.XML"
   
  failOnStderr: true
  displayName: 'Run Test-JSON'

- task: PublishTestResults@2
  displayName: 'Publish Test Results **/TEST-*.xml'
  inputs:
    failTaskOnFailedTests: true

- task: CopyFiles@2
  displayName: 'Copy policies and scripts'
  inputs:
    SourceFolder: '$(Build.SourcesDirectory)'
    Contents: |
     **/*
     !.git/**/*
    TargetFolder: '$(Build.ArtifactStagingDirectory)'

- task: PublishBuildArtifacts@1
  displayName: 'Publish Artifacts'
```

#### Release pipeline

Release pipeline (called just 'Release' in ADO interface) is automated way to push code to production. In our case that is policy publishing to the Azure. The pipeline is very simple and contains two stages each containing two tasks of type Azure PowerShell. 

![Release pipeline](\assets\img\2021\2021-06-17\ADO-Release-pipeline.png)

Each tasks is one of two PS scripts that define all logic of managing policies based on management group structure described in repo I wrote about above. 

1. `Set-AzMGPolicySetDeployment.ps1`. Script publishes policy initiative definition to a management group. This script accept as input parameter path for the root folder where management group structure is defined along with the policies. Each sub-folder under '_policies' directory is considered as policy initiative. Policy should be defined in policy.json file inside policy folder. Name of the folder should be equal to name of initiative defined in template (name property for type Microsoft.Authorization/policySetDefinitions). 
The script uses standard comandlets from Az module like New-AzManagementGroupDeployment, Get-AzManagementGroup. (When management groups were released only available programmic method to create deployment at MG level was to use REST API).
Parameters:
```powershell
    [Parameter(Mandatory = $True, HelpMessage = 'Specify local folder containing management group and policies')] [ValidateScript( { Test-Path $_ })] [String]$PoliciesRootFolder,
    [Parameter(Mandatory = $false, HelpMessage = 'Location for the deployment, westeurope by default')] [String]$DeploymentLocation = 'westeurope',
    [Parameter(Mandatory = $false, HelpMessage = 'Optional parameter for temporary management group for tests')] [String]$TestManagementGroup
```
2. `Set-AzMGPolicySetAssignment.ps1`. Script creates assignment to a management group. It has the same parameters as script for definition deployment. It uses `assignment.*.json` files inside policy set folders and use parameters from there to create assignment. Names of management group where policy will be assigned follow the 'assignment.' word of file names. For example, `assignment.MG1.json` contains assignment parameters for MG1 management group. **Script automatically sets policyDefinitionId property inside definitions.** It also assigns Contributor role for policy assignment where managed identity is used ("type": "SystemAssigned"). It uses Graph API to perform an assignment to a management group.

Each stage therefore consists of two tasks that first deploy and then assign policies.

![Release pipeline stage 1](\assets\img\2021\2021-06-17\ADO-Release-Pipeline1.png)

* Task **Publish Policy Set to Test/Prod environment**

```yaml
steps:
- task: AzurePowerShell@5
  displayName: 'Publish Policy Set to Test environment'
  inputs:
    azureSubscription: 'Azure-DevOps-Connector'
    ScriptPath: '$(System.DefaultWorkingDirectory)\$(Release.PrimaryArtifactSourceAlias)\drop\_scripts\Governance\Set-AzMGPolicySetDeployment.ps1'
    ScriptArguments: '-PoliciesRootFolder "$(System.DefaultWorkingDirectory)\$(Release.PrimaryArtifactSourceAlias)\drop" -TestManagementGroup "TestMG" -Verbose'
    azurePowerShellVersion: LatestVersion
```

* Task **Assign Policy Set to Test/Prod environment**

```yaml
steps:
- task: AzurePowerShell@5
  displayName: 'Assign Policy Set to Test environment'
  inputs:
    azureSubscription: 'Azure-DevOps-Connector'
    ScriptPath: '$(System.DefaultWorkingDirectory)\$(Release.PrimaryArtifactSourceAlias)\drop\_scripts\Governance\Set-AzMGPolicySetAssignment.ps1'
    ScriptArguments: '-PoliciesRootFolder "$(System.DefaultWorkingDirectory)\$(Release.PrimaryArtifactSourceAlias)\drop" -TestManagementGroup "TestMG" -Verbose'
    azurePowerShellVersion: LatestVersion
```

Depending on task `Set-AzMGPolicySetDeployment.ps1` or `Set-AzMGPolicySetAssignment.ps1` is invoked. For test stage additional parameter is used: *-TestManagementGroup "TestMG"*. 
Stage two is triggered upon approval after publishing and assigning to test environment is completed.

![approval after publishing](\assets\img\2021\2021-06-17\ADO-Release-approval.png)

Pipeline is enabled for continuous deployment and triggered after build pipeline deploys artifacts.

![Enable CD](\assets\img\2021\2021-06-17\AD-Enable-CD.png)

#### Testing it all together

Here is the sequence of publishing policy definitions and performing assignments by ADO:
* commit is performed to repository to directory representing root management group

![commit](\assets\img\2021\2021-06-17\git-push.png)

* commit triggers build pipeline checking policies and publishing artifacts (scripts and MG structure with policy files)

![build](\assets\img\2021\2021-06-17\ADO-build-complete.png)

* Upon completion release pipeline publish policies first to test environment and awaits for approval to continue for production

![release](\assets\img\2021\2021-06-17\ADO-trigger-release.png)

* In Azure deployment is created at management group level

![Azure deployment](\assets\img\2021\2021-06-17\Azure-deployment.png)

* assignment is performed for each MG defined in assignment file names

![Azure assignment](\assets\img\2021\2021-06-17\Azure-Assignment.png)

* Troubleshooting could be done using logs of build and release pipelines

![ADO logs](\assets\img\2021\2021-06-17\ADO-Logs.png)


#### Script sources
Scripts for both pipelines could be found in [Azure Policies repo of my GitHub](https://github.com/Astashin/Azure-Governance/tree/main/_scripts).

Azure DevOps project with repository and pipelines is made public and [accessible using a link](https://dev.azure.com/Astashin/Azure%20Policies).
