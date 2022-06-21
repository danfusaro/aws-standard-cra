---
marp: true
title: AWS Github CI CD Pipeline with CloudFormation
paginate: true
---

# The Goal

- Configure CI and CD pipelines using AWS
- When changes to our repo are merged, those changes are shown on a server
- Have a basic understanding of terminology
- Understand the steps and demystify the process
- Try it yourself

---

# Continuous Integration (CI) Pipeline

What does a "continuous integration (CI) pipeline" do?

- Reads code from repository
- Adds environment variables, set configs, use build parameters etc.
- Runs build, handles errors
- Report red/amber/greens status to release management tool, e.g. Github

**AWS CodeBuild is used to configure build settings.**

---

# Continuous Delivery (CD) Pipeline

What does a "continuous delivery (CD) pipeline" do?

- Ingest build artifact(s) from CI pipeline
- Put files into place
- Update server(s) and deploy

**AWS CodePipeline is used for to configure deployment settings.**

---

# Infrastructure as Code

CI/CD config and settings are "saved" somewhere so that it can be communicated, scaled, improved, etc.

**AWS CloudFormation is used to "save" our overall configuration**

---

# Let's do it

---

# Configure Github Access

1. Generate specific Github Access Token for AWS @ https://github.com/settings/tokens/new

   - Required to interop with our Git repo.
     - `repo` - Full control of private repositories
     - `admin: repo_hook` - Full control of repository hooks

---

2. Store token using AWS Secrets Manager
   - In AWS Console, search for "Secrets Manager"
   - Select secret type: "Other type of secret"
     - Key: `GITHUB_ACCESS_TOKEN`
     - Value: `[your personal access token]`
     - Click Next
   - Name it: `GITHUB_ACCESS` and click next until saved (Stored)
   - This can be used throughout all AWS applications

---

# Configure Web App

- Create `buildspec.yml`
  - Build specification file for CodeBuild
    - Directory structure
    - What to build, where to get it
  - Add basic build steps

```
version: 0.2

phases:
  build:
    commands:
      - echo Build started on `date`
```

- For more in-depth settings: https://docs.aws.amazon.com/codebuild/latest/userguide/build-spec-ref.html

---

# Configure CloudFormation

Let's create a template that will run our build (`pipeline.yaml`)

1. In AWS Console, go to "Cloud Formation"
2. Click Create Stack (with New Resources)
3. Create template in Designer

---

# Create YAML with CloudFormation Resource Designer

1. Choose "Template" tab (bottom)
2. Choose `YAML` format
3. Scroll to `Code Build` Resource type and drag and drop `Project` into designer pane
4. Remove all `Metadata` entries

---

# Policies

```
AWSTemplateFormatVersion: 2010-09-09
Description: CI/CD pipeline for github projects
Resources:
  CodeBuildServiceRole:
    Type: 'AWS::IAM::Role'
    Properties:
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          - Effect: Allow
            Principal:
              Service: codebuild.amazonaws.com
            Action: 'sts:AssumeRole'
      Policies:
        - PolicyName: root
          PolicyDocument:
            Version: 2012-10-17
            Statement:
              - Sid: CloudWatchLogsPolicy
                Effect: Allow
                Action:
                  - 'logs:CreateLogGroup'
                  - 'logs:CreateLogStream'
                  - 'logs:PutLogEvents'
                Resource: '*'
              - Sid: S3GetObjectPolicy
                Effect: Allow
                Action:
                  - 's3:GetObject'
                  - 's3:GetObjectVersion'
                Resource: '*'
              - Sid: S3PutObjectPolicy
                Effect: Allow
                Action:
                  - 's3:PutObject'
                Resource: '*'
              - Sid: ECRPullPolicy
                Effect: Allow
                Action:
                  - 'ecr:BatchCheckLayerAvailability'
                  - 'ecr:GetDownloadUrlForLayer'
                  - 'ecr:BatchGetImage'
                Resource: '*'
              - Sid: ECRAuthPolicy
                Effect: Allow
                Action:
                  - 'ecr:GetAuthorizationToken'
                Resource: '*'
              - Sid: S3BucketIdentity
                Effect: Allow
                Action:
                  - 's3:GetBucketAcl'
                  - 's3:GetBucketLocation'
                Resource: '*'
```

---

Creates a Role to run the pipelines. `AssumeRolePolicyDocument` gives any service the ability to assume a role and follows the spec in `Version`. Values copied and pasted from [AWS Codebuild User Guide.](https://docs.aws.amazon.com/codebuild/latest/userguide/setting-up.html#setting-up-service-role) - see `create-role.json` and `put-role-policy.json`. `CodeCommitPolicy` is removed because GitHub is being used for source control in our example.

---

# Auth Artifact

Define our Credential/Auth artifact

```
CodeBuildSourceCredential:
    Type: 'AWS::CodeBuild::SourceCredential'
    Properties:
      AuthType: PERSONAL_ACCESS_TOKEN
      ServerType: GITHUB
      Token: '{{resolve:secretsmanager:GITHUB_ACCESS:SecretString:GITHUB_ACCESS_TOKEN}}'
```

---

# Project Configuration

```
  CodeBuildProject:
    Type: 'AWS::CodeBuild::Project'
    Properties:
      Name: !Ref 'AWS::StackName'
      ServiceRole: !GetAtt CodeBuildServiceRole.Arn
      Source:
        Type: GITHUB
        Location: 'https://github.com/danfusaro/aws-standard-cra.git'
        BuildSpec: buildspec.yml
        Auth:
          Type: OAUTH
          Resource: !Ref CodeBuildSourceCredential
      Artifacts:
        Type: NO_ARTIFACTS
      Triggers:
        Webhook: true
        FilterGroups:
          - - Type: EVENT
              Pattern: 'PULL_REQUEST_CREATED, PULL_REQUEST_UPDATED'
            - Type: BASE_REF
              Pattern: !Sub ^refs/heads/main$
      Environment:
        Type: LINUX_CONTAINER
        ComputeType: BUILD_GENERAL1_SMALL
        Image: aws/codebuild/standard:4.0
```

---

CodeBuildProject config includes a reference to our created Service Role and references the GitHub project, build file, and auth setting.

`Artifacts` is set to `NO_ARTIFACTS` temporarily but this will specify what files are included after build.

`Triggers` specifies which events will start a build. This is configured specifically against Git syntax.

`Environment`: Basic Linux container, compute type and the Docker image provided by CodeBuild

---

## Final Steps

1. Validate your file with the "Validate template" button (Top, checkbox)
2. Copy the resulting text file as `pipeline.yaml` in your web application root.
3. Save this configuration in AWS with the "Create Stack" (cloud icon) in Designer
4. Name and configure your stack, e.g. "aws-standard-cra-ci-cd" , acknowledge creation of IAM resources
5. Create stack, cross fingers.

View the "Resources" tab to see what was created

---

## CodeBuild Project

- Navigate to "CodeBuild" in the AWS Console
- Note linked primary repository from GitHub
- In the "Build details" tab, note the overall config and Primary service webhook events
- Click the link to see the Webhook config in GitHub

---

## Validate webhook

1. Create a `develop` branch and push a change
2. Submit a pr against `main`
3. Wait for status to show "AWS CodeBuild" message

---

## Add branch security

This ensures that our CI pipeline passes before allowing a merge.

1. In Settings, add a branch protection rule
2. Check "Reuire status checks to pass before merging"
3. Search for "AWS CodeBuild ..."

---

# Add configurable settings (optional)

In order to make this pipeline config truly flexible, we can requiren some inputs
