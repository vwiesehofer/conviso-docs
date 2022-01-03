---
id: azure-pipelines-graph
title: Azure Pipelines Graph Mode
sidebar_label: Azure Pipelines Graph Mode
---

<div style={{textAlign: 'center'}}>

![img](../../static/img/azure-pipelines.png)

</div>

## Introduction

The Azure Pipelines is a CI/CD module of the [Azure Devops](https://aex.dev.azure.com/) platform. Through this module, it is possible to create automation routines with various tasks that are available on Azure's marketplace. Currently, the integration with Conviso consists of Bash-type tasks. Among the tasks are: the FlowCLI command line interface ([FlowCLI available at PyPi](https://pypi.org/project/conviso-flowcli/)), running SAST, running deploy for code review, and also the two actions together.

## Requirements

In order for the experience with Conviso's services to be complete, it is necessary to meet all the requirements below:

1. Hosted Agent Pool (Ubuntu 18.04 or higher) with Docker installed or Agent Cloud Azure;

2. External access (may be limited to Conviso's registry for SAST, Dockerhub and Conviso Platform).

## First Steps

Given an Azure Devops project, to create a Welcome Pipeline you can follow the steps below:

1. At the DevOps Project root, click at **Pipelines**;

2. At the upper right menu, click at **New Pipeline**;

3. Select the **Use the classic editor to create a pipeline without YAML** option;

4. At **Select your Repository** step, select the platform where your code is hosted, the ropository and the branch for pipeline execution and click at **Continue**;

5. Select the **Start with an Empty Job** option;

6. Rename the **Agent Job 1** to Conviso Agent, selecting Agent Pool option as **Azure Pipelines** and Agent Specification option as **ubuntu-18-04**;

7. At Conviso Agent, click at the **+** icon to add a new task;

8. Add a **Bash** type task, rename the Display Name to **Install Conviso FlowCLI** and modify its type to **Inline**;

9. At the script field, add the code snippet presented below:

```yml
echo "Installing Conviso FlowCLI..."
sudo pip3 install conviso-flowcli
flow --version
```

10. Click at **Save & Queue**. The pipeline execution will begin in a few moments.

In this first block of **Bash** task, we carry out the installation of the tool and check if everything is fine. Then, we'll define how other blocks will be after further actions.

## Code Review Deployment

Before proceeding, we recommend reading the following [guide](../guides/code-review-strategies) to understand the different strategies/approaches for deploying Code Review.

After choosing the strategy used to send deploys to Code Review, it is possible to create a specific pipeline for this action, as well as integrate with other existing pipelines. The requirements for executing this functionality are the settings of the ```FLOW_API_KEY``` variables previously configured in the desired pipeline variables and the ```FLOW_PROJECT_CODE``` variable (identified as the Project Key at Conviso Platform) that can be defined in each of the pipelines.

Below are the code snippets that must be inserted in the blocks for each type of strategy: 

**With TAGS, sorted by timestamp**

```yml
flow -k $(FLOW_API_KEY) deploy create with tag-tracker sort-by time --project-code $(FLOW_PROJECT_CODE)
```

**With TAGS, sorted by versioning-style**

```yml
flow -k $(FLOW_API_KEY) deploy create with tag-tracker sort-by versioning-style --project-code $(FLOW_PROJECT_CODE)
```

**Without TAGS, sorted by GIT Tree**

```yml
flow -k $(FLOW_API_KEY) deploy create with values --project-code $(FLOW_PROJECT_CODE)
```

## SAST

In addition to deploying for code review, it is also possible to integrate a SAST-type scan into the development pipeline, which will automatically perform a scan for potential vulnerabilities, treated in Conviso Platform as findings.

The requirements for running the job are the same as already practiced: ```FLOW_API_KEY``` and ```FLOW_PROJECT_CODE``` defined as environment variables for the Agent Pool.

```yml
flow sast run --project-code $(FLOW_PROJECT_CODE)
```

In the above pipeline, we didn't use any options to the ```flow sast run``` command. In this case, the default behavior is to perform the analysis of the entire repository. This is because the default values used for the ```--start-commit``` and ```--end-commit``` options use first commit and current commit (HEAD), respectively.

Alternatively, we can specify the diff range manually. In the example below, we scan between the current commit and the immediately previous one on the current branch:

```yml
flow sast run --start_commit `git rev-parse @~1` --end-commit $(Build.SourceVersion) --project-code $(FLOW_PROJECT_CODE)
```

## Getting everything together: Code Review + SAST Deployment

The SAST analysis can be complementary to the code review carried out by the professional at Conviso, even serving as input for the analyst. The job below will perform the deploy for code review of the code and will use the same diff identifiers to perform the SAST analysis, forming a complete solution in the pipeline. An example of a complete pipeline with both solutions can be seen in the snippet below:

```yml
flow -k $(FLOW_API_KEY) deploy create -f env_vars with values > created_deploy_vars
source created_deploy_vars
flow -k $(FLOW_API_KEY) sast run --project-code $(FLOW_PROJECT_CODE)
```

## Troubleshooting
If authentication is not performed even when loading the ```FLOW_API_KEY``` variable, make sure it is provided as environment variables for all tasks that use the FlowCLI.