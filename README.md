# Automate Cedar policy validation with AWS Developer Tools

[Cedar](https://www.cedarpolicy.com/) is an open-source language for writing authorization policies and making authorization decisions based on those policies. AWS security services including [AWS Verified Access](https://aws.amazon.com/verified-access/) and [Amazon Verified Permissions](https://aws.amazon.com/verified-permissions/) use Cedar for defining policies. Cedar supports schema declaration for the structure of entity types in those policies and [policy validation](https://docs.cedarpolicy.com/policies/validation.html) using that schema. The solution outlined in this repo implements a build pipeline using [Developer Tools on AWS](https://aws.amazon.com/products/developer-tools/) which validates the Cedar policy files against a schema and runs a suite of tests isolating the Cedar policy logic.

## Scenario

This repository extends the hypothetical photo sharing application from the Cedar policy language in action workshop. That app allows users to organize their photos into albums and share them with groups of users. Figure 1 visually illustrates entities from the photo application.

![Visual illustration of entities from the photo application](/PhotoApplication.png)
Figure 1 - Photo Application entities

## Local environment setup

Start by cloning this repo. Before committing this source code to a CodeCommit repository, the test suite can be run locally.  This allows you to shorten the feedback loop using different options.

Option 1: [Install Rust](https://www.rust-lang.org/tools/install) and compile the Cedar CLI binary

1. Install Rust using the rustup tool

    `curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y"`

1. Compile the Cedar CLI (version 2.4.2) binary using cargo

    `cargo install cedar-policy-cli@2.4.2`

1. Run the test runner script which tests authorize requests using the Cedar CLI

    `cd policystore/tests && ./cedar_testrunner.sh`

Option 2: Run the CodeBuild agent

Locally evaluate the `buildspec.yml` inside a CodeBuild container image using the `codebuild_build.sh` script from [aws-codebuild-docker-images](https://github.com/aws/aws-codebuild-docker-images) with the following parameters:

`./codebuild_build.sh -i public.ecr.aws/codebuild/amazonlinux2-x86_64-standard:5.0 -a .codebuild`

## Pipeline setup

Provision the services used in the pipeline using CloudFormation.

1. Create a new CloudFormation stack from the template:

    ```bash
    aws cloudformation deploy \    
    --template-file template.yml \    
    --stack-name cedar-policy-validation \    
    --capabilities CAPABILITY_NAMED_IAM
    ```

1. Wait for the message **Successfully created/updated stack**.

## CodePipeline invocation

Commit source code to a CodeCommit repository, then configure and invoke CodePipeline.

1. Add an additional remote named “codecommit”, pointing to the CodeCommit repository created by CloudFormation, to the repository previously cloned.  The `CedarPolicyRepoCloneUrl` stack output is the HTTPS Clone URL.  Replace with `CedarPolicyRepoCloneGRCUrl` to use the HTTPS (GRC) Clone URL when [connecting to AWS CodeCommit with git-remote-codecommit](https://docs.aws.amazon.com/codecommit/latest/userguide/setting-up-git-remote-codecommit.html).

    `git remote add codecommit $(aws cloudformation describe-stacks --stack-name cedar-policy-validation --query 'Stacks[0].Outputs[?OutputKey==`CedarPolicyRepoCloneUrl`].OutputValue' --output text)`

1. Push the code to the CodeCommit repository

    `git push codecommit main`

1. This starts a pipeline run.  Check progress with this command:

    ```bash
    aws codepipeline get-pipeline-execution \
    --pipeline-name cedar-policy-validation \
    --pipeline-execution-id $(aws codepipeline list-pipeline-executions --pipeline-name cedar-policy-validation --query 'pipelineExecutionSummaries[0].pipelineExecutionId' --output text) \
    --query 'pipelineExecution.status' --output text
    ```

The build installs Rust and compiles the Cedar CLI.  After about four minutes, the pipeline execution status turns **Succeeded**.

## Project structure

The `policystore` directory contains one Cedar policy per `.cedar` file. The Cedar schema is defined the `cedarschema.json` file. A `tests` subdirectory contains a `cedarentities.json` file representing application data, with its subdirectories (e.g. Album JaneVacation) representing the test suites.  The test suite directories contain individual tests inside their `ALLOW` and `DENY` subdirectories, each with one or more JSON files containing the authorization request to be evaluated against the policy set.

The `cedar_testrunner.sh` script runs the Cedar CLI to perform a `validate` command for each .cedar file against the Cedar schema, outputting either PASS or ERROR.  The script also performs an `authorize` command on each test file, outputting either PASS or FAIL depending on if the results match the expected authorization decision.

## Clean-up

To avoid ongoing costs and clean-up resources deployed into your AWS account, perform these steps in the AWS Console:

1. Navigate to **Amazon S3**, select the bucket starting with “cedar-policy-validation-codepipelinebucket”, and **Empty** the bucket
1. Navigate to **CloudFormation**, select the “cedar-policy-validation” stack, and **Delete**
1. Navigate to the **CodeBuild**, then **Build History**, filter by “cedar-policy-validation:”, select all, and **Delete builds**

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.

