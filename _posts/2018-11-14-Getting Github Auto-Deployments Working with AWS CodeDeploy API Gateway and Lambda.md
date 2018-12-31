---
title:  "Getting Github Auto-Deployments Working with AWS CodeDeploy, API Gateway, and Lambda"
tags: [github, aws, apigateway, serverless]
---

NOTE: This assumes that you already have a functional and tested CodeDeploy process built out, and a branch that you want to push with CodeDeploy (i.e. AddCodeDeploy in this example).

### Create an IAM Policy to Allow Access to CodeDeploy
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "codedeploy:GetDeploymentConfig",
            "Resource": "arn:aws:codedeploy:us-west-2:YOUR_ACCOUNT_ID:deploymentconfig:*"
        },
        {
            "Effect": "Allow",
            "Action": "codedeploy:RegisterApplicationRevision",
            "Resource": "arn:aws:codedeploy:us-west-2:YOUR_ACCOUNT_ID:*"
        },
        {
            "Effect": "Allow",
            "Action": "codedeploy:GetApplicationRevision",
            "Resource": "arn:aws:codedeploy:us-west-2:YOUR_ACCOUNT_ID:*"
        },
        {
            "Effect": "Allow",
            "Action": "codedeploy:CreateDeployment",
            "Resource": "arn:aws:codedeploy:us-west-2:YOUR_ACCOUNT_ID:*"
        }
    ]
}
```

### Create an IAM Policy to Allow Access to Lambda and CloudWatch for Logging
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "logs:CreateLogGroup",
            "Resource": "arn:aws:logs:us-west-2:YOUR_ACCOUNT_ID:*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": [
                "arn:aws:logs:us-west-2:YOUR_ACCOUNT_ID:log-group:/aws/lambda/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "lambda:InvokeFunction"
            ],
            "Resource": [
                "*"
            ]
        }
    ]
}
```

### Create Lambda function. This will execute when you hit the API Gateway, and will call CodeDeploy to Start a New Build

1. AWS > Lambda > Create function > Author from scratch
2. Name = TestFunc
3. Runtime = Node.js 6.10
4. Environment variables - Add a new entry called Secret, with a value of A_LONG_RANDOM_STRING_OF_CHARACTERS_THAT_WE'LL_ALSO_CONFIGURE_ON_THE_GITHUB_SIDE

The code. This will take a message from a Github webhook, and if it's a commit from the branch "AddCodeDeploy", it'll kick off a build in AWS CodeDeploy.

```
exports.handler = (event, context, callback) => {
    var AWS = require('aws-sdk');
    var crypto = require('crypto');
    
    //grab the Secret environment variable
    var secret = process.env.Secret;
    //hash the event.body - we'll use this to validate the message we got is really from Github
    var hash = crypto.createHmac('sha1', secret).update(JSON.stringify(event.body));
    //get the digest of the message from Github
    var digest = 'sha1=' + hash.digest('hex');
    
    //check to see that the digest and the header we got from Github match - indicating it's a legit message from Github
    if (digest === event.headers["X-Hub-Signature"]) {
        console.log("Legit message from Github.");
        
        var codedeploy = new AWS.CodeDeploy({apiVersion: '2014-10-06'});
    
        if (event.body.ref === 'refs/heads/AddCodeDeploy') {
            console.log('Received a push on the AddCodeDeploy branch. Pushing a CodeDeploy deployment...');
        
            var params = {
                applicationName: 'MyCodeDeployApp',
                autoRollbackConfiguration: {
                    enabled: false
                },
                deploymentGroupName: 'TEST',
                revision: {
                    gitHubLocation: {
                        commitId: event.body.head_commit.id,
                        repository: 'myOrganization/MyApp'
                    },
                    revisionType: 'GitHub',
                },
                fileExistsBehavior: 'OVERWRITE'
            }
        
            codedeploy.createDeployment(params, function(err, data) {
                if (err) {
                    console.log(err, err.stack); // an error occurred
                } else {
                    console.log(data); // successful response
                }
            });
        }
    } else {
        console.log("Dropping message - Potentially bogus message from Github");
    }
    
    //respond to the web request
    callback();
};
```

### Add Our API Gateway that Fronts our Lambda Code

1. AWS > API Gateway > New API
2. API name = TestAPI
3. Action > Create Method > POST - Click the checkbox
4. Integration type = Lambda Function
5. Lambda Region = us-west-2
6. Lambda Function = TestFunc
7. Actions > Deploy API
8. Deployment Stage = stage, Deploy

By default, API Gateway does not pass headers to Lambda functions. We'll want the headers passed in because we need that to validate the message we receive actually came from GitHub.

### Add Support to Push Headers Received on API Gateway to our Lambda Function

1. AWS > API Gateway > TestAPI
2. Click our POST method > Integration Request
3. Click the Body Mapping Templates section to expand
4. Request body passthrough = "When there are no templates defined (recommended)"
5. Add mapping template
6. Enter "application/json" for the Content-Type, and click the Check mark to save.
7. Enter the following in the textbox that appears below under Generate Template. This will take the headers that are sent to API Gateway and turn them into JSON accessible by our Lambda function
```
{
    "method": "$context.httpMethod",
    "body" : $input.json('$'),
    "headers": {
        #foreach($param in $input.params().header.keySet())
        "$param": "$util.escapeJavaScript($input.params().header.get($param))"
        #if($foreach.hasNext),#end
        #end
    }
}
```
8. Click Save
9. Back at the top - Actions > Deploy API
10. Deployment stage = test. When the deployment is complete, take note of the URL for our API - we'll need this to configure the Github side of things. It should be something like https://RANDOM_GUID.execute-api.us-west-2.amazonaws.com/test

At this point, our AWS code should be set up to receive webhook requests from Github. Let's set up the Github Side of things.

### Configure GitHub to Call API Gateway with a WebHook Request Upon Builds

1. Log into Github, and go to the Settings page for your repository
2. Click the Webhooks tab > Add webhook
3. Payload URL = Enter the API Gateway URL we received above
4. Content Type = application/json
5. Secret = THE_SAME_LONG_RANDOM_STRING_OF_CHARACTERS_THAT_WE_CONFIGURED_ON_THE_LAMBDA_SIDE
6. Leave the event as "Just the push event", and click Add webhook

At this point, you should be able to push a new commit to your AddCodeDeploy branch.

### Testing

1. cd to your repo
2. notepad index.html (or a file in your repo)
3. git commit -am "Testing CodeDeploy Github webhook"
4. git push origin AddCodeDeploy

If everything worked correctly, you should see a new build triggered immediately in CodeDeploy! :D

