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
   - https://us-east-1.console.aws.amazon.com/secretsmanager/home?region=us-east-1#!/home or search for "Secrets Manager"
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
