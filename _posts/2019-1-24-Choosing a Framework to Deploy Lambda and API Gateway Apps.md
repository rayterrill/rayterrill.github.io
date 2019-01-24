---
title:  "Choosing a Framework to Deploy Lambda and API Gateway Apps"
tags: [aws, apigateway, serverless]
---

At our org, we're pushing hard into converting some of our existing applications to utilize serverless technologies, and specifically Lambda + API Gateway. While we normally use Terraform to deploy our infrastructure into AWS, the amount of code required to get Lambda + API Gateway integrations deployed was really a showstopper for us - just absolutely brutal. So we embarked on looking at a number of tools to see if we could find a better way.

### ClaudiaJS

We initially looked at ClaudiaJS, due to it's reputation for being very simple to get started. We were excited to get a few things deploy pretty quickly using the proxyAPI functionality, but quickly ran into issues where we needed to go out and make manual changes to the deployed code in AWS after-the-face because of limitations in that model (specifically things like requiring api keys, etc). We pivoted to the claudia-api-builder model, but ended up running into limitations there as well.

Pros:
- Very simple to get started
- Pretty easy to take existing code and migrate it to use Claudia
- Multiple deployment methods to fit your goal
- Developers were very responsive to questions

Cons:
- The documentation is not great. A number of caveats and design decisions are documented, but not easily accessible. We did eventually find them, but only after we had stumbled across a limitation.
- The framework is fairly opinionated. There is a one-to-one relationship between a ClaudaJS function and an API - no ability to map individual functions to methods within an API. We found this very limiting.
- The proxyAPI functionality does not include the ability to require API keys. This meant manual configuration outside of Claudia in order to complete a deployment.
- No mechanism to handle custom domains/base path mappings - must be handled outside the tool
- Configuration appears to be command-line parameter based, as opposed to using a config file. This makes it a little difficult to get things going with junior devs and ensure repeatable deployments.

### Serverless Framework

After running into the issues with ClaudiaJS, we pivoted our efforts over to Serverless Framework. As with ClaudiaJS, we were able to pretty quickly get things deployed using the examples. We also were able to get some more advanced things going with a little trial and error. At this point, we've been able to deploy our code to PROD, and we have not encountered any insurmountable issues.

Pros:
- Pretty simple to get started
- Easy to take existing code and migrate it to use Serverless Framework
- Multiple deployment methods to fit your goal
- Provides both simple configuration options and the ability to override and configure the more advanced pieces for deployments
- Documentation is pretty detailed for simple things

Cons:
- The documentation is pretty good for simple things, but for some of the advanced things we've had to do a bunch of googling to see how others are handling it

### Conclusion

In the end, we've elected to move forward with Serverless Framework, as it appears to support the most options going forward, but still is relatively simple to bootstrap new developers onto the framework.
