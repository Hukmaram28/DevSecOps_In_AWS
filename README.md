# DevSecOps in AWS Cloud and Istio Mesh Service

## Overview

DevSecOps has become a very common word in the devops So we will be going to integrate various open source security tools in our pipeline which we created in the previous project [Link to the previous project](https://github.com/Hukmaram28/AWS_EKS_Vault_Chaos_Engineering/)
We will complete the project in 3 sections:

PART-1: Introduction DevSecOps

PART-2: Implementation of DevSecOps pipeline with git-secrets, CAST highligh and SAST stages.

PART-3: Implementation of DevSecOps pipeline with EKS Deployment, DAST and RASP stages.

## Prerequisites

- An AWS Free Tier account.
- Understanding of DevOps pipeline setup and deploying applications on an EKS cluster. Please refer to my previous project [here](https://github.com/Hukmaram28/AWS_EKS_Vault_Chaos_Engineering/).

## Introduction to DevSecOps

DevOps combines software development and IT operations to help companies deliver new features and services faster.

DevSecOps goes further by adding security into DevOps. This approach allows you to make secure and compliant updates quickly, using automation for consistent operations.

![DevSecOps](./images/DevSecOps.png)

A complete DevSecOps pipeline is essential for a successful software workflow. This includes:

- Continuous Integration (CI)
- Continuous Delivery and Deployment (CD)
- Continuous Testing
- Continuous Logging and Monitoring
- Auditing and Governance
- Operations

Finding vulnerabilities early in the software development process can save a lot of money on fixing issues. Automating this process also helps deliver changes faster.

## Security Vulnerability Scanning Stages

To find security vulnerabilities at different stages, organizations can use various tools and services in their DevSecOps pipelines. Setting up these tools and combining the vulnerability findings can be difficult to do from scratch.

AWS offers services and tools to make this easier. You can integrate both AWS and third-party tools into your DevSecOps pipeline, and AWS also provides services to aggregate security findings.

Below, we will discuss different third-party tools and AWS services.

## Secrets Scanning

Secrets scanning is the process of finding sensitive information embedded in code repositories. There are many tools available for this, and they are often better than building your own solution. These tools have useful features like scanning version control history, custom signatures, and multiple reporting formats. An engineer can choose the tool that fits their context and requirements the best.

Some examples of secrets scanning tools are:

- Git-secrets
- Trufflehog
- detect-secrets

## SCA/SAST (Static Application Security Testing)

Static code analysis, also known as static code review, is the process of finding bad coding practices, potential vulnerabilities, and security flaws in software source code without running it. This method helps teams catch code bugs or vulnerabilities that manual code reviews and compilers often miss.

Static code analysis provides a quick, automated feedback loop for detecting defects that could become serious problems if not addressed.

Besides checking code styles, static code analysis is also used for static application security testing (SAST).

Some examples of SCA/SAST tools are CAST highligh, Trivy, Synk, Anchor, and CoreOS Clair.

## DAST (Dynamic Application Security Testing)

While SAST tools scan the code for vulnerabilities, DAST tools scan the application once it’s running. DAST tools simulate attacks on the application to find security issues, much like a hacker would. This approach provides more relevant findings with substantial evidence of the vulnerabilities.

DAST results are easier for developers to understand because they show the consequences of a vulnerability, helping developers gauge the severity of the risk.

DAST scanners are a good first step in integrating security into DevOps (DevSecOps). They simplify vulnerability scanning for developers and make it easier to understand the security risks. DAST tools categorize vulnerabilities into High, Medium, and Low, and can be seamlessly integrated into your CI/CD pipeline.

Some effective DAST tools available in the market are:

- OWASP ZAP
- Crashtest Security
- Arachni

## RASP (Runtime Application Self-Protection)

RASP is a technology that runs on a server and activates when an application is running. It detects attacks in real time by analyzing the app's behavior and the context of that behavior. This allows RASP to protect the application from malicious input or actions immediately, without human intervention.

RASP integrates security directly into the running application on the server. It intercepts all calls from the app to a system to ensure they are secure and validates data requests directly within the app. Both web and non-web applications can be protected by RASP. The technology does not affect the design of the app, as RASP's detection and protection features operate on the server where the app is running.

One example of a RASP tool for EKS clusters is Falco.

## DevSecOps Tool Stack

We will be using following tools for the EKS deployment process in AWS:

- **Git-Secret**: Checks for sensitive information being committed to code repositories.
- **Hadolint**: A Dockerfile linter that validates inline bash scripts, written in Haskell.
- **Checkov**: Checkov is a static code analysis tool that scans infrastructure as code (IaC) or deployment files for misconfigurations that could lead to security or compliance problems
- **Anchore**: Tool for Software Composition Analysis (SCA) and Static Application Security Testing (SAST).
- **ECR Scanning**: An in-built feature that uses Clair for internal scanning.
- **OWASP ZAP**: A tool for Dynamic Application Security Testing (DAST).
- **Falco**: Provides Runtime Application Self-Protection (RASP).

Let's begin the implementation of these tools in our exisitng CICD pipeline.

We will create various codeBuild stages for all the security tools in our pipeline. To demonstrate, we are going to ingerate all these tools in our exisiting CI/CD pipeline for web microservice which is a reactJs application. The sample Dockerfile, buildspec.yaml and helm templates are present under `web` folders.
We will create a seprate buildspec YML for each tool.

First of all we need to enable AWS config and security hub so please do so.

AWS-Config: AWS Config Record and evaluate configurations of your AWS resources. AWS Config provides a detailed view of the resources associated with your AWS account, including how they are configured, how they are related to one another, and how the configurations and their relationships have changed over time.

Security-Hub: AWS Security Hub provides a consolidated view of your security status in AWS. Automate security checks, manage security findings, and identify the highest priority security issues across your AWS environment.

## Git-Secrets stage

In the pipeline, the second stage is the Git Secrets Check. During this stage, the GitHub repository is scanned by the git-secrets tool. It scans the entire repository for sensitive information such as credentials. If any sensitive information is found, the CodeBuild process fails since the git-secrets scan command returns a non zero exit code (`git secrets --scan -r .`). If no sensitive information is found, the build succeeds.

The build file for git-secrets stage is preset at ./web/git-secrets.yml

![git-secrets](./images/git-secrets.png)

![git-secrets-2](./images/git-secrets-2.png)

## Hadolint and Checkov Stage

In this stage, the Hadolint tool scans the Dockerfile for any syntax issues. Hadolint is a smart Dockerfile linter that helps you create best practice Docker images. It parses the Dockerfile into an Abstract Syntax Tree (AST) and applies rules on top of it. Additionally, it uses Shellcheck to lint the Bash code inside RUN instructions.

Checkov is a static code analysis tool that scans infrastructure as code (IaC) files for misconfigurations that could lead to security or compliance problems. Checkov can scan various IaC file types, including:

- Terraform (for AWS, GCP, Azure, and OCI)
- CloudFormation (including AWS SAM)
- Azure Resource Manager (ARM)
- Serverless framework
- Helm charts
- Kubernetes
- Docker

The build file for hadolint and checkov stage is preset at ./web/hado-checkov.yml

![hadolint-checkov](./images/hadolint-checkov.png)

Below is the output of the Checkov tool.

![output](./images/checkov.png)

## Anchor Stage

