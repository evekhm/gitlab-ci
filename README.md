# GitLab CI Modules

##Table of Contents

## Table of Contents
- [Overview](#overview)
- [Setup](#setup)
  * [GitLab Projects](#gitlab-projects)
  * [CI/CD Environments](#ci-cd-environments)
- [Architecture Diagrams](#architecture-diagrams)
- [Build](#build)
  * [Image Tagging Logic](#image-tagging-logic)
  * [Application CI file](#application-ci-file)
- [Deploy](#deploy)
  * [Triggered from Upstream](#triggered-from-upstream)
  * [Triggered from Pipeline](#triggered-from-pipeline)
  * [Deploy CI file](#deploy-ci-file)
  * [Deploy script](#deploy-script)
- [GitLab Limitations](#gitlab-limitations)

## Overview

The goal of this CI/CD infrastructure is to leverage Gitlab integration with GCP and allow single Project to be used for different GCP deployments addressing the various personas needs.
The offered templates (modules) allow simplifying CI/CD while accounting for the complex micro-service architecture, when different services can be developed independently and kept in different projects. 


Use Cases:
1) As a *Customer Engineer*  I want to have an easy way to deploy Solution into my own GCP Project environment and use it to demo to a customer, while having full control over the GCP project.
2) As a *Developer*, I want to be able to work on the feature which spans across multiple projects and have automated CI/CD to deploy images that either belong to the feature and were modified by me, or the released versions. 
3) As a *QA engineer*, I want to be able to have MR related feature projects bee deployed into the test environment.
4) As a *Sales Person*, I want to have a stable demo environment for the customer presentations.
5) ...

## Setup
### GitLab Projects
We will demonstrate the usage of the plug-in templates with the following example of GitLab Projects setup:

**Services**: (different Projects, each has Container Registry enabled):
- ApplicationA - service A
- ApplicationB - service B
- ApplicationC - service C

**Manifest Project**:
- DeployApplications -  GCP deployment project, containing GKE manifests yaml file for the applications.

**Gitlab Templates**:
- [This]() Project, offering templates for building and deploying applications into the different GCP environment, while following the lifecycle of the solution development. 

### CI/CD Environments <a name="ci-cd-environments"></a>
CI/CD covers Following Environments:
- *development* environment(s) - personalized GCP environment(s), can be created/setup and used per developer.
- *test* environment - pre-configured GCP environment used for testing before manually releasing application images or changes to the project setup or deployment flow.
  The namespace separation (named after the feature branch) will be used during the deployment.
- *demo* - running stable demo using released images and main branch for the CI/CD deployment.

## Architecture Diagrams

![](img/gcp-gitlab-cicd.png)

![](img/feature-development.png)

![](img/downstream-trigger.png)

![](img/commit-pipeline.png)

![](img/merge-rq-pipeline.png)

## Build
### Image Tagging Logic

Image Tagging and URL depends on the source branch or whetehr even is a merge request, in the following way:

- Main branch             ~>   released:latest  
- Feature branch         ~>   feature-name/qa:latest
- Merge Request (feature to main) ~>   feature-name/mr:latest

### Application CI file 

![](img/build-image.png )

```shell
variables: # TO be passed downstream
  DEPLOY_PROJECT: '<path-to-deployapplications>'

include:
- project: '<path-to-gitlab-ci>'
  file: '/.gitlab/ci/.build.gitlab-ci.yml'
- project: '<path-to-gitlab-ci>'
  file: '/.gitlab/ci/.deploy-trigger.gitlab-ci.yml'

stages:
- build
- vars 
- deploy
```
## Deploy

### Triggered from Upstream
When deployment is triggered from the Upstream project, *gitlab-ci deploy Job* will call the  deploy.sh script from Manifest Project and pass required parameters:

```shell
APPLICATION=$APPLICATION IMAGE=$IMAGE KUBE_NAMESPACE=$NAMESPACE ENVIRONMENT=$ENVIRONMENT bash ${CI_PROJECT_DIR}/deploy.sh
```

### Triggered from Pipeline
When triggered from the pipeline, deployment iterates through the registered applications and uses latest image based on the following logic:
- If it is a main branch ~> uses released images
- If it is a feature branch ~> checks if there is a latest image for the feature branch of the application, or falls back to the released image
- If merge request ~> checks if there are any mr images in the container registry of the Application.

For an Application to be properly registered within the deployment automation following two steps need to be done:
- APPLICATION_NAMESPACE needs to be defined
- a folder needs to be added:  `applications/<application-name>` with the application-name matching GitLab project directory name from the URL (same as `CI_PROJECT_NAME`).
  *  The name of the directory for the project. For example if the project URL is gitlab.example.com/group-name/project-1, CI_PROJECT_NAME is project-1.

### Deploy CI file 
```shell
include:
  project: <path-to-gitlab-ci>
  file: /.gitlab/ci/.gitlab/ci/deploy.gitlab-ci.yml

variables:
  # Path to the Namespace of the Application(s) Projects with micro-services.
  # Assumption: All of them are under the same hierarchy and same namespace.
  APPLICATION_NAMESPACE: <path-to-application-namespace>

  # Could Point to the same AGENT
  KUBE_CONTEXT_DEMO: "<path-to-gitlab-agent-for-demo-env>"
  KUBE_CONTEXT_TEST: "<path-to-gitlab-agent-for-test-env>"
  KUBE_CONTEXT_DEV: "<path-to-gitlab-agent-for-dev-env>"

stages:
- deploy
- destroy

```

### Deploy script
The service specific logic needs to be added for the deployment to be executed, when deploy.sh is called by the `deploy` Job.

We are having the following structure for each of the registered applications inside applications/<application_name>:
- k8s  - contains required yaml files for the deployment
- gcp  - contains  logic to pass through parameters such as IMAGE

####Examples:

deploy.sh
```shell
DIR="$( cd "$( dirname "${BASH_SOURCE[0]}" )" && pwd )"

if [ -n "$APPLICATION" ]; then
  APPLY_SCRIPT="${DIR}"/applications/"${APPLICATION}"/gcp/apply.sh
  if [ -f "${APPLY_SCRIPT}" ] ; then
    bash "$APPLY_SCRIPT"
  else
    echo  "Error: Invalid path to apply deployment $APPLY_SCRIPT"
  fi
else
  echo "Error, APPLICATION is not set"
fi

```
apply.sh
```shell
DIR="$( cd "$( dirname "${BASH_SOURCE[0]}" )" && pwd )"
echo "***** Applying  $APPLICATION deployment  *****"
K8S="$DIR/../k8s/"
sed 's|__IMAGE_TAG__|'"$IMAGE"'|g;' "$K8S/deployment.sample.yaml" > "$K8S/deployment.yaml"
kubectl apply -f "$K8S/deployment.yaml" --namespace="$KUBE_NAMESPACE"
kubectl apply -f "$K8S/service.yaml" --namespace="$KUBE_NAMESPACE" 
```

deployment.sample.yaml
```shell
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    application: applicationa
  name: applicationa
spec:
  replicas: 1
  selector:
    matchLabels:
      application: applicationa
  template:
    metadata:
      labels:
        application: applicationa
    spec:
      containers:
      - image: __IMAGE_TAG__
        name: applicationa
        ports:
        - containerPort: 80
      imagePullSecrets:
      - name: regcred
      restartPolicy: Always
```


## GitLab Limitations

### Downstream Trigger
The following code, does not work when variable for the project is not defined as static, but needs to be substituted. To overcome this limitation, we are generating yaml file using `deploy-template.yml` on the fly.
```shell
  trigger:
    project: $DEPLOY_PROJECT
```

Must be:
```shell
  trigger:
    project: 'my-project-path'
```

### Pre-defined variables issue
Following variable is supposed to be set when Pipeline is triggered, however this is not the case. Instead, we introduce and use our own flag `PIPELINE_TRIGGERED`.

```shell
CI_PIPELINE_TRIGGERED
```

### Late Variable Binding
Rules and Variables have proved not to work too well, when variables are dynamic.
Because of that extra code needed to be added and in general usong of dynamic variables should be avoided. 

Because of that, I cannot easily extend rules and decide to skip execution of the job, if for-example KUBE_CONTEXT is empty, which would be an easy way for a user to decide, for example, not to deploy MR into test and skip that step.

The code snippet below would always skip the Job, because $KUBE_CONTEXT is not substituted as expected.
```shell
deploy_dev:
  environment: development
  rules:
  - if: '$KUBE_CONTEXT == null'
    when: never
```
