# API Gateway with Multiple Deployments

Our team has decided to embark on a conversion process, where we're planning to migrate a NodeJS + MongoDB API running in EC2 and surfaced via API Gateway to a new Serverless NodeJS using Lambda, DynamoDB, and API Gateway.

As a part of the initial feasibility work, I was able to convert our NodeJS API over to DynamoDB with a minimum of hassle (probably good for another blog post), and we were ready for some testing.

### Trying to Grok API Gateway Stages and the Limitations

Once I had the NodeJS API code converted over to DynamoDB (in a development branch in Github), I was ready to publish the API for some internal testing. I thought I understand API Gateway stages relatively well, until I went to actually do this. It appears that API Gateway stages are basically snapshots of how the API looked at a particular time, but there appears to be no **easy** way to have two stages do vastly different things while retaining the ability to make changes to both stages if needed - for example have one stage that points to an EC2 instance via VPC link, and another stage that points to a lambda function. You can actually do this, but it looks like that actually means changing the API definition to accomodate the new functional - then making a snapshot of the API to represent that new functionality, with no easy way to go back and make modifications to the "old API" if needed.

### A Path Forward

We use the Custom Domain Names function of API Gateway to map a custom domain onto our API, along with the Base Path Mapping function (currently we're using the root of the mapping "/"). As a result of the limitations described above and the need to maintain two separate API deployments for the duration of the migration, we're planning to switch to using a stage-like syntax going forward for the Base Path Mapping - effectively mapping a v1 base path mapping to our old EC2/MongoDB/VPC Link API, and a v2 mapping to our new Serverless/DynamoDB API.

![APIGatewayMultipleStages]({{ site.url }}/assets/APIGatewayMultipleStages.PNG)
