---
title:  "AWS CodePipeline Deploy to S3 with Terraform"
tags: [aws, terraform]
---

AWS recently released support for deploying to S3 with CodePipeline [https://aws.amazon.com/about-aws/whats-new/2019/01/aws-codepipeline-now-supports-deploying-to-amazon-s3/](https://aws.amazon.com/about-aws/whats-new/2019/01/aws-codepipeline-now-supports-deploying-to-amazon-s3/).

This fits in perfectly with the work we've been doing to move things to Serverless, but we wanted to make sure we keep things consistent across our accounts, which we currently do with Terraform.

#### Build the supporting pieces CodePipeline needs to deploy code (a iam policy, role, and the policy attachment to that role):
```
#build the policy for the codepipeline service role
resource "aws_iam_policy" "AWSCodePipelineServiceRole-us-west-2-github" {
  name        = "AWSCodePipelineServiceRole-us-west-2-github"
  path        = "/"

  policy = <<EOF
{
    "Statement": [
        {
            "Action": [
                "iam:PassRole"
            ],
            "Resource": "*",
            "Effect": "Allow",
            "Condition": {
                "StringEqualsIfExists": {
                    "iam:PassedToService": [
                        "cloudformation.amazonaws.com",
                        "elasticbeanstalk.amazonaws.com",
                        "ec2.amazonaws.com",
                        "ecs-tasks.amazonaws.com"
                    ]
                }
            }
        },
        {
            "Action": [
                "codecommit:CancelUploadArchive",
                "codecommit:GetBranch",
                "codecommit:GetCommit",
                "codecommit:GetUploadArchiveStatus",
                "codecommit:UploadArchive"
            ],
            "Resource": "*",
            "Effect": "Allow"
        },
        {
            "Action": [
                "codedeploy:CreateDeployment",
                "codedeploy:GetApplication",
                "codedeploy:GetApplicationRevision",
                "codedeploy:GetDeployment",
                "codedeploy:GetDeploymentConfig",
                "codedeploy:RegisterApplicationRevision"
            ],
            "Resource": "*",
            "Effect": "Allow"
        },
        {
            "Action": [
                "elasticbeanstalk:*",
                "ec2:*",
                "elasticloadbalancing:*",
                "autoscaling:*",
                "cloudwatch:*",
                "s3:*",
                "sns:*",
                "cloudformation:*",
                "rds:*",
                "sqs:*",
                "ecs:*"
            ],
            "Resource": "*",
            "Effect": "Allow"
        },
        {
            "Action": [
                "lambda:InvokeFunction",
                "lambda:ListFunctions"
            ],
            "Resource": "*",
            "Effect": "Allow"
        },
        {
            "Action": [
                "opsworks:CreateDeployment",
                "opsworks:DescribeApps",
                "opsworks:DescribeCommands",
                "opsworks:DescribeDeployments",
                "opsworks:DescribeInstances",
                "opsworks:DescribeStacks",
                "opsworks:UpdateApp",
                "opsworks:UpdateStack"
            ],
            "Resource": "*",
            "Effect": "Allow"
        },
        {
            "Action": [
                "cloudformation:CreateStack",
                "cloudformation:DeleteStack",
                "cloudformation:DescribeStacks",
                "cloudformation:UpdateStack",
                "cloudformation:CreateChangeSet",
                "cloudformation:DeleteChangeSet",
                "cloudformation:DescribeChangeSet",
                "cloudformation:ExecuteChangeSet",
                "cloudformation:SetStackPolicy",
                "cloudformation:ValidateTemplate"
            ],
            "Resource": "*",
            "Effect": "Allow"
        },
        {
            "Action": [
                "codebuild:BatchGetBuilds",
                "codebuild:StartBuild"
            ],
            "Resource": "*",
            "Effect": "Allow"
        },
        {
            "Effect": "Allow",
            "Action": [
                "devicefarm:ListProjects",
                "devicefarm:ListDevicePools",
                "devicefarm:GetRun",
                "devicefarm:GetUpload",
                "devicefarm:CreateUpload",
                "devicefarm:ScheduleRun"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "servicecatalog:ListProvisioningArtifacts",
                "servicecatalog:CreateProvisioningArtifact",
                "servicecatalog:DescribeProvisioningArtifact",
                "servicecatalog:DeleteProvisioningArtifact",
                "servicecatalog:UpdateProduct"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "cloudformation:ValidateTemplate"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "ecr:DescribeImages"
            ],
            "Resource": "*"
        }
    ],
    "Version": "2012-10-17"
}
EOF
}

#builds the service role for codepipeline
resource "aws_iam_role" "AWSCodePipelineServiceRole-us-west-2-github" {
  name = "AWSCodePipelineServiceRole-us-west-2-github"

  assume_role_policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "codepipeline.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF
}

#attach the policy to the service role
resource "aws_iam_role_policy_attachment" "AWSCodePipelineServiceRole-us-west-2-github" {
  role       = "${aws_iam_role.AWSCodePipelineServiceRole-us-west-2-github.name}"
  policy_arn = "${aws_iam_policy.AWSCodePipelineServiceRole-us-west-2-github.arn}"
}
```

#### Build the S3 bucket for our artifacts:
```
#build the bucket for our codepipeline artifacts
resource "aws_s3_bucket" "codepipeline-us-west-2-myartifacts" {
  bucket = "codepipeline-us-west-2-myartifacts"
  acl    = "private"
}
```

#### Finally, Build the CodePipeline Itself:
```
resource "aws_codepipeline" "github" {
  name     = "github-pipeline"
  role_arn = "${aws_iam_role.AWSCodePipelineServiceRole-us-west-2-github.arn}"

  artifact_store {
    location = "${aws_s3_bucket.codepipeline-us-west-2-myartifacts.bucket}"
    type     = "S3"
  }

  stage {
    name = "Source"

    action {
      name             = "Source"
      category         = "Source"
      owner            = "ThirdParty"
      provider         = "GitHub"
      version          = "1"
      output_artifacts = ["SourceArtifact"]

      configuration {
        Owner  = "MyGithubOrg"
        Repo   = "MyRepo"
        Branch = "master"
        OAuthToken = "${var.GITHUB_TOKEN}"
      }
    }
  }

  stage {
    name = "Deploy"

    action {
      name            = "Deploy"
      category        = "Deploy"
      owner           = "AWS"
      provider        = "S3"
      input_artifacts = ["SourceArtifact"]
      version         = "1"

      configuration {
        BucketName = "myappname.mydomain.com"
        Extract = "true"
      }
    }
  }
}
```
