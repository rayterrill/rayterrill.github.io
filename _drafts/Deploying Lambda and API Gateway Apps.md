---
title:  "Deploying Lambda and API Gateway Apps."
tags: [aws, apigateway, serverless]
---

At our org, we're pushing hard into converting some of our existing applications to utilize serverless technologies, and specifically Lambda + API Gateway. While we normally use Terraform to deploy our infrastructure into AWS, the amount of code required to get Lambda + API Gateway integrations deployed was really a showstopper for us - just absolutely brutal. So we embarked on looking at a number of tools.

### ClaudiaJS

We initially looked at ClaudiaJS, due to it's reputation for being very simple to get started. We were excited to get a few things deploy pretty quickly using the proxyAPI functionality, but quickly ran into issues where we needed to go out and make manual changes to the deployed code in AWS after-the-face because of limitations in that model (specifically things like requiring api keys, etc). We pivoted to the claudia-api-builder model, but ended up running into limitations there as well.

Pros:
- Very simple to get started
- Pretty easy to take existing code and migrate it to use Claudia
- Multiple deployment methods to fit your goal

Cons:
- The documentation is not great. A number of caveats and design decisions are documented, but not easily accessible. We did eventually find them, after we had stumbled across a limitation.
- The framework is very opinionated. There is a one-to-one relationship between a ClaudaJS function and an API - no ability to map functions to individual functions to individual methods within an API. We found this very limiting.
- The proxyAPI functionality does not include the ability to require API keys.
- No mechanism to handle custom domains/base path mappings - must be handled outside the tool

### Serverless Framework
