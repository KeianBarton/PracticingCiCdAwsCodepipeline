# Practicing CI/CD with AWS CodePipeline

https://app.pluralsight.com/library/courses/practicing-cicd-aws-codepipeline

# Building and Deploying with CodePipeline

## Overview

AWS CodePipeline:
- Automated CI service
- Workflow definition
- Triggered from source control
- Low barrier to entry gor AWS teams
- Cheap ($1 per pipeline excluding processing time or S3 storage)
- Easy setup for AWS applications

CodePipeline Structure

Each pipeline consists of:
- Configuration options (name identifier, service role, S3 bucket for artifact store, encryption key)
- Minimum of 2, maxiumm of 10 **stages**
- Each stage contains 1 to 50 **actions**
- State carries through **input/output** artifacts between actions (stored in S3 bucket) - e.g. Action 1, Output: SourceArtifact, Action 2, Input: SourceArtifact, Output: BuildArtifact

Example Pipeline:

| Stage        | Actions                  | Inputs/Outputs         |
|--------------|--------------------------|------------------------|
| Source stage | Source Action            | Output: SourceArtifact |
| Build stage  | Build Action Test Action | Input: SourceArtifact  |
| Stage        | Action                   |                        |

First action must contain one source action (and cannot contain any other action type)

Action Types:
- **Source Action Type** (can only appear in 1st stage, pulls in source code)
- **Build Action Type**
- **Test Action Type**
- **Deploy Action Type**
- **Approval Action Type** (manual approval required to continue execution)
- **Invoke Action Type** (invoke an AWS Lambda function)

Ways to create pipelines:
- AWS console
- CloudFormation template to build a pipeline (recommended and easy to recreate)

## Demo

Course Demo Project Code:

github.com/ryanmurakami/codepipeline-hbfl

Preparing for a pipeline:

- CodeCommit -> Create Repository
- IAM -> Create New Group (e.g. hbfl-code-committers) and give access to repo
- Add users to group
- For each user, you can generate HTTPS Git credentials for AWS CodeCommit
- After pushing first commit, you can run the following to create an elastic beanstalk application and environment:

```aws cloudformation create-stack --stack-name hbfl-eb --template-body file://.aws/hbfl-ep-app.template --capabilities CAPABILITY_IAM``` (update line 12 of ./aws/hbfl-eb-app.template with the correct solution stack with Node.js for your region)

Creating/editing a pipeline (steps):

1. **Pipeline settings**: CodePipeline -> Create new pipeline (and new service role if you need it)
2. **Source stage**: Choose source provider and repo name; Change detection options (CloudWatch Events is default and recommended, versus legacy period checking for changes)
3. **Build stage**: Select Create project (e.g. hbfl-build); link it to a buildspec YAML file
4. **Deploy stage** Add action group and add a deploy action pointing at the BuildArtifact input artifact

Select **Release Change** to trigger a deployment

You can click the Elastic Beanstalk link in your Deploy Stage after deploying to see the environments and live website running

Pipeline Execution in Detail:
- Each execution has a unique ID
- Executions run until they succceed, fail or are stopped
- Pipelines can run multiple executions; stages only one at a time
- Stages lock when executing; subsequent executions must wait

Running Tests in CodePipeline:

- Add a Test Action under the Build Action using the Input BuildArtifact
- Point to a (non-default) buildspec test yml file
- If you wanted Integration tests, you could add an integration stage with a Deploy Action and Integration Test Action

Manual Approvals in CodePipeline:

- Demo uses example to send SMS to notify if approval is needed (skipping adding notes):
```aws cloudformation create-stack --stack-name hbfl-sns --template-body file://.aws/hbfl-sns.template --parameters ParamerKey=SNSEndpoint,ParameterValue=<phone_#>```
- In the Approval Action you can add SNS topic ARN (used for SMS example above) or a URL for review

## Using CodePipeline with CloudFormation

Reasons to use CloudFormation:

- Easy to reproduce infrastructure in other regions, accounts
- Can track infrastructure changes in source control
- Quick and easy removal of resources

Template is in the demo code in `hbfl-pipeline.template` (RunOrder within an action determines order - use the same number for parallel running of actions)

You can run the template in command line using:

```aws cloudformation create-stack --stakc-name hbfl-pipeline --template-body file://.aws/hbfl-pipeline.template --capabilities CAPABILITY_NAMED_IAM```

# Advanced CodePipeline Practices

## Invoking Lambdas from CodePipeline

We can use the Lambda invocation Action Type

Lambda Invocation Configuration Options:
- Input artifacts
- User parameters (pre-configured string data passed to the Lambda function)

Lambda functions must use the AWS SDK to explicitly tell the pipeline execution to continue. It won't continue on its own (even if the Lambda has finished running):
- ```putJobSuccessResult``` - marks Lambda invocation action result as Success and continues execution
- ```putJobFailureResult``` - marks Lambda invocation action result as Failure and stops execution
- ```startPipelineExecution```
- ```getPipelineState```

Lambda functions need additional IAM permissions to report back to CodePipeline (service role used by the Lambda function needs CodePipeline access) e.g.

```javascript
const AWS = require('aws-sdk'); // Included with all Lambdas
exports.handler = (event, context) => {
    const codePipeline = new AWS.CodePipeline();
    const jobId = event['CodePipeline.job'].id;
    codePipeline.putJobSuccessResult({ jobId }, (err, data) => {
        if (err) {
            context.fail();
        } else {
            context.succeed();
        }
    })
}
```

Then add an action to (e.g. the Source) stage point to the Lambda created.

## Stage Transitions in CodePipeline

Stage transitions are:

- Connections between stages that can be disabled.
- Good to avoid unnecessary or disruptive builds while developing.
- Don't use for situations where manual approvals are a better fit (e.g. delay deployment before human intervention)

## Monitoring CodePipeline Changes with CloudWatch Events

CloudWatch Event rules can be configured for the following events:
- Pipeline Changes
- Stage Changes (e.g. Cancelled, Failed, Started, Succeeded)
- Action Changes
- CodePipeline API Calls

CloudWatch -> Events -> Rules to configure -> CodePipeline Action Execution State Change -> Specific state FAILED

Set the Target to invoke when an event matches your event pattern (e.g. SNS topic to send notifications)

Use the Pipeline ARN (available from Pipeline settings) to narrow failures to just action failures on our specific pipeline:

```javascript
{
    "source": [
        "aws.codepipeline"
    ],
    "detail-type": [
        "CodePipeline Action Execution State Change"
    ],
    "resources": [ "arn:aws:codepipeline..... arn " ],
    "detail": {
        "state": [
            "FAILED"
        ]
    }
}
```

## Using Notification Rules in CodePipeline

Easier than CloudWatch to set up to send notifications if (e.g. action has failed)

In CodePipeline, select Nofiy -> Manage notification rules