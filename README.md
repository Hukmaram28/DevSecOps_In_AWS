# DevSecOps in AWS Cloud and Istio Mesh Service

## Overview

DevSecOps has become a very common word in the devops So we will be going to integrate various open source security tools in our CICD pipeline which we created in the previous project. [Link to the previous project](https://github.com/Hukmaram28/AWS_EKS_Vault_Chaos_Engineering/)

We will be using following tools for the EKS deployment process in AWS:

- **Git-Secret**: Checks for sensitive information being committed to code repositories.
- **Hadolint**: A Dockerfile linter that validates inline bash scripts, written in Haskell.
- **Checkov**: Checkov is a static code analysis tool that scans infrastructure as code (IaC) or deployment files for misconfigurations that could lead to security or compliance problems
- **Anchore**: Tool for Software Composition Analysis (SCA) and Static Application Security Testing (SAST).
- **ECR Scanning**: An in-built aws ECR feature that uses Clair for image scanning.
- **OWASP ZAP**: A tool for Dynamic Application Security Testing (DAST).
- **Falco**: Provides Runtime Application Self-Protection (RASP).
- **Security hub**: To aggragate all the reports from various security tools in one place.

# Architecture

![Architecture](./images/architecture.png)


## Introduction to DevSecOps

DevOps combines software development and IT operations to help companies deliver new features and services faster.

DevSecOps goes further by adding security into DevOps. This approach allows us to make secure and compliant updates quickly, using automation for consistent operations.

![DevSecOps](./images/DevSecOps.png)

Finding vulnerabilities early in the software development process can save a lot of money on fixing issues. Automating this process also helps deliver changes faster.

## Security Vulnerability Scanning Stages

To find security vulnerabilities at different stages, organizations can use various tools and services in their DevSecOps pipelines. Setting up these tools and combining the vulnerability findings can be difficult to do from scratch.

AWS offers services and tools to make this easier. You can integrate both AWS and third-party tools into your DevSecOps pipeline, and AWS also provides services to aggregate security findings such as securitiyhub.

Below, we will discuss different third-party tools and AWS services.

## Secrets Scanning

Secrets scanning is the process of finding sensitive information embedded in code repositories. There are many tools available for this, and they are often better than building own solution. These tools have useful features like scanning version control history, custom signatures, and multiple reporting formats. An engineer can choose the tool that fits their context and requirements the best.

Some examples of secrets scanning tools are:

- Git-secrets
- Trufflehog
- detect-secrets

## SCA/SAST (Static Application Security Testing)

Static code analysis, also known as static code review, is the process of finding bad coding practices, potential vulnerabilities, and security flaws in software source code without running it. This method helps teams catch code bugs or vulnerabilities that manual code reviews and compilers often miss.

Static code analysis provides a quick, automated feedback loop for detecting defects that could become serious problems if not addressed.

Besides checking code styles, static code analysis is also used for static application security testing (SAST).

Some examples of SCA/SAST tools are CAST highligh, Trivy, Synk, Anchore, and CoreOS Clair.

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

Let's begin the implementation of these tools in our exisitng CICD pipeline.

We will create various codeBuild stages for various security tools in our pipeline. To demonstrate, we are going to integrate all these tools in our exisiting CI/CD pipeline for web microservice which is a reactJs application. The sample Dockerfile, buildspec.yaml and helm templates are present under `web` folder.

We will create a seprate buildspec YML for each tool.

First of all we need to enable AWS config and security hub so please do so by visiting aws console.

AWS-Config: AWS Config Record and evaluate configurations of your AWS resources. AWS Config provides a detailed view of the resources associated with your AWS account, including how they are configured, how they are related to one another, and how the configurations and their relationships have changed over time.

Security-Hub: AWS Security Hub provides a consolidated view of your security status in AWS. Automate security checks, manage security findings, and identify the highest priority security issues across your AWS environment.

## Git-Secrets stage

In the pipeline, the second stage is the Git Secrets Check. During this stage, the GitHub repository is scanned by the git-secrets tool. It scans the entire repository for sensitive information such as credentials. If any sensitive information is found, the CodeBuild process fails since the git-secrets scan command returns a non zero exit code (`git secrets --scan -r .`). If no sensitive information is found, the build succeeds.

The build file for git-secrets stage is preset at `./web/buildspec-gitsecrets.yml`

![git-secrets](./images/git-secrets.png)

![git-secrets-2](./images/git-secrets-2.png)

## Hadolint and Checkov Stage (SCA)

In this stage, the Hadolint tool scans the Dockerfile for any syntax issues. Hadolint is a smart Dockerfile linter that helps you create best practice Docker images. It parses the Dockerfile into an Abstract Syntax Tree (AST) and applies rules on top of it. Additionally, it uses Shellcheck to lint the Bash code inside RUN instructions.

Checkov is a static code analysis tool that scans infrastructure as code (IaC) files for misconfigurations that could lead to security or compliance problems. Checkov can scan various IaC file types, including:

- Terraform (for AWS, GCP, Azure, and OCI)
- CloudFormation (including AWS SAM)
- Azure Resource Manager (ARM)
- Serverless framework
- Helm charts
- Kubernetes
- Docker

The build file for hadolint and checkov stage is preset at `./web/buildspec-hado-checkov.yml`

![hadolint-checkov](./images/hadolint-checkov.png)

Below is the output of the Checkov tool.

![output](./images/checkov.png)

## Anchore Stage (SAST)

In this stage, the Dockerfile will be built and scanned by the Anchore tool to detect any vulnerabilities present in the Docker image.
The Anchore tool will save the vulnerability results in a JSON format file. Using a Lambda function, the contents of this file will be uploaded to SecurityHub, where we can find detailed information about High, Medium, and Low level vulnerabilities. The report is also stored in s3 for future reference. These reports will be helpful to the development team in remediating the vulnerabilities.
The Docker image will also be uploaded to the ECR repository.

The lambda function used in the stage are available at `./lambda-functions`. Create this lambda function manually before triggering the pipeline. Create and assign proper IAM policies to the lambda IAM role to allow access to the S3 and securityhub.

The build file for Anchore is `./web/buildspec-anchore.yml`

![alt text](./images/anchore_pipeline.png)

![Anchore](./images/anchore.png)

![Lambda](./images/anchore_lambda.png)

The anchore results are published to security hub and stored in S3 using lambda function.

![anchore_securityhub](./images/anchore_securityhub.png)

![anchore_s3](./images/anchore_s3.png)

## ECR scanning

AWS ECR uses clair to scan docker images, it can be manually triggered as well it can be enabled in the ECR settings for regular automatic scans.

To scan during the image build, we can trigger a scan using aws cli command in a build stage and publish the results to securityhub using a lambda function.

Example ECR build file present at `./web/buildspec-ecr.yml`

![ECR stage](./images/ecr-stage.png)

![ecrscan](./images/ecrscan.png)

The ECR scan results are published to secuity hub and stored in S3 using the lambda function!

![ecr_securityhub](./images/ecr_securityhub.png)

![ecr_s3](./images/ecr_s3.png)

## EKS Deployment Stage

This stage is already done in the previous project. Link the the previous project [Here](https://github.com/hukmaram28/AWS_EKS_Vault_Chaos_Engineering)

We are using helm to package and deploy the application to EKS cluster.

The build file for deployment is `./web/buildspec-deployment.yml`

![eks_deploy](./images/eks_deploy.png)

Here is the output of helm deployment from codeBuild stage:

![helm_logs](./images/helm_logs.png)

result of `kubectl get all -n dev`

![deployment](./images/helm_eks.png)

## OWASP-Zap Scan Stage (DAST)

In this stage, the OWASP ZAP tool will take the load balancer URL from the previous stage as input and scan it for any vulnerabilities. The tool will generate a JSON format output file, which will then be processed by a Lambda function to send the content to AWS SecurityHub. The development team can find the vulnerability reports in SecurityHub and address them as necessary.

If the OWASP ZAP tool detects any warnings or failures, the CodeBuild job will fail and display the detected issues. If a warning is not a vulnerability, it can be ignored by specifying it in a configuration file, which is then passed as an argument to the OWASP ZAP scan command. This setup uses a baseline ZAP scan for the process.

All output files generated during the scan will be stored in an S3 bucket. This ensures that the reports and data are safely archived and accessible for future reference or further analysis.

The codeBuild file for owasp-zap tool is `./web/buildspec-owasp.yml`

![owasp_stage](./images/owasp_stage.png)

![owasp_output](./images/owasp_output.png)

The report is also published to security hub and stored in s3 using lambda function.

![owasp_securityhub](./images/owasp_securityhub.png)

![owasp_s3](./images/owasp_s3.png)

## Falco Stage (RASP)

Finally, we install the Falco tool in the EKS cluster (either using IaC or manually). Falco is a cloud-native runtime security project and serves as the Kubernetes threat detection engine. It leverages syscalls, which are essential components of its core functionality. Falco parses the syscalls that occur between the application and the kernel, checking them against predefined rules and generating alerts when any rule violations are detected.
We use Helm charts to install Falco in the cluster as a daemon set, alongside Fluent Bit which helps in collecting and pushing the logs to cloudwatch. Additionally, an AWS CloudWatch log group is created to store all alerts.

If there is any unexpected behavior or intrusion in the EKS cluster, it will be logged in CloudWatch alert logs. From these logs, we can create metric filters and an SNS topic to receive alerts via email. This setup ensures that any security incidents within the cluster are promptly detected and communicated, allowing for timely response and mitigation.

We can deploy falco to our cluster using helm. The instructions are available in folder `falco`

Once installed we will be able to see cloudWatch logs generated by falco.

![falco_cmd](./images/falco_command.png)

![falco k8s](./images/falso_k8s.png)

### References

https://aws.amazon.com/devops/

https://owasp.org/www-project-zap/

https://falco.org/blog/intro-k8s-security-monitoring/

https://anchore.com/container-vulnerability-scanning

https://hub.docker.com/r/hadolint/hadolint

https://hub.docker.com/r/bridgecrew/checkov

https://docs.aws.amazon.com/codepipeline/latest/userguide/welcome.html
