# Picky CICD
This repository contains the assets for Github actions which includes workflows and actions which can be re-used by other projects.

There are two types of workflows currently in this repository:
- build-and-release - for building artifacts and uploading them to an AWS CodeArtifact repository.

![Build and Release Diagram](./assets/build-and-release.png)

- cicd - for creating a docker image, uploading it to AWS ECR and deploying it on AWS ECS

![CICD Diagram](./assets/cicd.png)

## How to use it
To use a CICD workflow inside your own project, add the following directories and file inside your project at the root path: `.github/workflows/cicd.yml`

```yaml
---
name: {project-name}-cicd

on: [push]

jobs:
  cicd:
    uses: go-picky/picky-microservices/.github/workflows/{workflow-name}.yaml
    with:
      name: {project-name}
    secrets: inherit
```

Note: pick the workflow which is most suitable for the project. They can be found inside this repository under `.github/workflows/`


The workflows require secrets in order to run and access all of the external services outside of GitHub.
Secrets can be configured at the repository level under `https://github.com/{organisation}/{repository-name}/settings/secrets/actions`

The required secrets are:

Workflow:
- gradle-build-and-release
  - `AWS_ACCESS_KEY_ID`
  - `AWS_DOMAIN` - domain that CodeArtifact is hosted on
  - `AWS_SECRET_ACCESS_KEY`
  - `CLUSTER_ARN` - identifier for ECS cluster
- gradle-cicd
  - `AWS_ACCESS_KEY_ID`
  - `AWS_DOMAIN`
  - `AWS_SECRET_ACCESS_KEY`


## AWS ECS
Projects using the [AWS deploy action](https://github.com/go-picky/picky-cicd/blob/main/actions/aws/deploy/action.yaml) expects a task definition file defined at the path: `ecs/{development|production}.task-def.yaml`. The content can be defined using the following task definition template:
```yaml
# Abstract the values into a template
executionRoleArn: arn:aws:iam::{account_id}:role/{role_name}
containerDefinitions:
  - name: media-service-container
    image: {account_id}.dkr.ecr.{region}.amazonaws.com/{image_name}
    environment:
      - name: CONFIG_PATH
        value: config/development.toml
    cpu: 0
    portMappings:
      - containerPort: 8080
        hostPort: 8080
        protocol: tcp
    essential: true
    logConfiguration:
      logDriver: awslogs
      options:
        awslogs-group: /ecs/{task_definition_name}
        awslogs-region: {region}
        awslogs-stream-prefix: ecs
family: {task_definition_name}
taskRoleArn: arn:aws:iam::{account_id}:role/{role_name}
networkMode: awsvpc
runtimePlatform:
  operatingSystemFamily: LINUX
requiresCompatibilities:
  - EC2
  - FARGATE
cpu: 512
memory: 1024
```

## Directory structure
```
.
├── .github
│   └── workflows                       (1)
│       └── gradle-build-and-release.yaml
│       └── gradle-cicd.yaml
└── actions                             (2)
    ├── aws                             (3)
    ├── docker                          (4)
    └── gradle                          (5)
```

1. Workflows for different types of projects e.g. Java, Node etc  - see [reusable workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows) for more information
1. Actions which are referenced by the workflows - see [composite actions](https://docs.github.com/en/actions/creating-actions/creating-a-composite-action) for more information
1. Actions related to AWS operations
1. Actions related to Docker operations
1. Actions related to Gradle operations

## Considerations

### Cluster URL
Currently the deploy stages are hardcoded to deploy to the development cluster of ECS and a placeholder registry on ECR.
This will need to be changed once we are ready to use this CICD for production deploys or when we have a proper development envioronment for testing.

### Upload Limits
Github imposes an upload limit on the output artifacts for jobs. As a workaround, the the build and release jobs have been combined into one stage. This should be more gracefully moving forward.
