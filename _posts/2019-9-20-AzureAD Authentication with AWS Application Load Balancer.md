---
title:  "AzureAD Authentication with AWS Application Load Balancer"
tags: [ad, aws, azuread]
---

AWS' Application load balancer supports OIDC authentication, but I couldn't find a single document that shows how to configure this to work with AzureAD auth.

### AzureAD Configuration

1. Create a new application in the Application Registrations section of AzureAD
2. Configure the Redirect URI as both your load balancer hostname, as well as your hostname + "/oauth2/idpresponse" (Ex: https://myapp.mydomain.com and https://myapp.mydomain.com/oauth2/idpresponse)
3. Create a secret for your application and save this for later - you'll need this for the AWS configuration
4. On the API permissions screen, click the button to "Grant admin consent" for your org. No additional permissions are required at this point other than the default User.Read Microsoft Graph permission.

### AWS Configuration

We manage our AWS infrastructure with Terraform.

The biggest problem I had here was discovering the correct endpoints for the OIDC configuration. Eventually I found these are all available in the .well-known configuration, which should be available at: https://login.microsoftonline.com/YOUR_TENANT_ID/v2.0/.well-known/openid-configuration

Here's the configuration for the ALB load balancer rule with AzureAD authentication. Substitute in your Tenant ID, Client ID, and Secret.

```
resource "aws_lb_listener_rule" "ALBTest" {
  listener_arn = data.terraform_remote_state.ECS.outputs.aws_lb_listener_arn

  action {
    type = "authenticate-oidc"

    authenticate_oidc {
      authorization_endpoint = "https://login.microsoftonline.com/YOUR_TENANT_ID/oauth2/v2.0/authorize"
      client_id              = "YOUR_CLIENT_ID"
      client_secret          = "YOUR_CLIENT_SECRET"
      issuer                 = "https://login.microsoftonline.com/YOUR_TENANT_ID/v2.0"
      token_endpoint         = "https://login.microsoftonline.com/YOUR_TENANT_ID/oauth2/v2.0/token"
      user_info_endpoint     = "https://graph.microsoft.com/oidc/userinfo"
    }
  }

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.ALBTest.arn
  }

  condition {
    field  = "host-header"
    values = ["myapp.mydomain.io"]
  }
}
```
