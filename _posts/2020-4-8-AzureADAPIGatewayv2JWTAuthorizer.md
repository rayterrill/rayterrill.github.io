---
title:  "AzureAD Authentication with AWS API Gateway v2 JWT Authorizers"
tags: [aws, azuread]
---

AWS' API Gateway v2 (aka HTTP APIs) launched in December 2019, and came with a built-in ability to add JWT authorizers to endpoints. We use AzureAD as our Auth vendor, so I've been waiting for a chance to try this out. Finally got an opportunity.

I'm assuming you have a functional AzureAD application - I have an existing React app - so I'm just going to cover the pieces I needed to add/modify to get the AzureAD integration working.

### AzureAD Configuration

This ended up being one of the most difficult parts.

#### Create an App Registration for your API

1. Create a new application in the Application Registrations section of AzureAD
2. Add any permissions your API might need in the API Permissions tab
3. Go to the Expose an API tab, and click Add a Scope
    * Give your scope a name, admin consent name and description, and click Add Scope.
4. Once your app registration has been configured - take note of the Application/Client ID and Scope name you created above - you'll need these for the API Gateway configuration
    
#### Modify your Application Registration to Grant Access to the API

1. Location your application in the Application Registrations section of AzureAD
2. Under API Permissions, click Add a Permission
3. Under either My APIs or APIs My Organization Uses, find the App Registration created for the API in the above section, check the box to add the scope, and click Add Permissions. If you marked Admin Consent as required, click the Grant admin consent button or get a Global Admin to perform the consent process to grant the necessary permissions.

At this point, your application should be able to request a token to call the AzureAD API.

### AWS Configuration

#### Build your Lambda Function

I had an existing Lambda function that I used for this, but if necessary, build a Lambda function and ensure it works and returns something to the caller so you're able to test things out.

#### Build the API Gateway v2 Configuration

1. In API Gateway, click APIs on the left nav, and then Create API
2. Click the Build button under HTTP API
3. On the Create an API screen, click Add Integration, choose Lambda, and pick the correct Region, as well as your Lambda function. Enter a name for your API, then click Next to continue
4. I set my resource path to / for simplicity, but by default it appears to choose the Lambda function's name. Whatever you choose is fine, you'll just need to remember this to test things out with your client. Click Next to continue.
5. Leave the Configure Stages section alone (configured for Auto-deploy), and click Next to continue.
6. Click Create to create the API Gateway configuration

#### Build your JWT Authorizer

1. Once your API Gateway configuration has been created, click Authorization in the left nav
2. Click the VERB for your newly created route - by default it should be ANY - and then click the button for Create an attach an authorizer
3. Give your Authorizer a name, and configure your Authorizer for AzureAD, then click Create and Attach
    * Identity Source: $request.header.Authorization
    * Issuer: Even though the AWS documentation says this should be coming from the well-known metadata endpoint, which can be found at https://login.microsoftonline.com/<YOUR AZUREAD TENANT GUID>/v2.0/.well-known/openid-configuration, I found that the value here did not match what was in my issued tokens. I had to set the issuer to https://sts.windows.net/<YOUR AZUREAD TENANT GUID>/
    * Audience: api://<API APP Registration Application/ClientID>, for example api://00000000-0000-0000-0000-000000000000

#### Testing with Postman

At this point, you should be able to test your API with Postman. You'll have to obtain an access token good for your API.

I use https://www.npmjs.com/package/react-aad-msal in my React applications, but I ran into an issue where even when I specified the scope for my API Application in my authProvider.js file, the token I was getting was still only good for the MS Graph API. Decoding the Access Token with JWT.io showed an invalid signature - apparently access tokens for MS Graph API include a nonce value that makes the token not validate correctly against the public key (https://github.com/AzureAD/microsoft-authentication-library-for-js/issues/521) - learn something new everyday. Big shout out to Eric Johnson@AWS for helping me track this down.

I ended up adding some code to my React project to dump an access token to the console, and used react-aad-msal's ability to call into the underlying msal library to make this happen:
```
var tokenRequest = {
  scopes: ["api://<MY API's APP REGISRATION/CLIENTID>/<MY API's SCOPE>"]
};

//call in and get a token for our api
let token = await props.provider.acquireTokenSilent(tokenRequest)
console.log(token.accessToken);
```

Once you have an access token valid for your API (you can pretty easily decode and check this with jwt.io), you should be able to use Postman to access the API - just set the Authorization to Bearer token and paste your access token into the token section.

#### Adding CORS Support

To be able to use this in a browser-based app which was my goal, there's some additional configuration required.

1. In API Gateway, click CORS in the left-hand nav, configure the following settings, then click Save to save your settings.
    * Access-Control-Allow-Origin: Enter any origins which will need access to the API
    * Access-Control-Allow-Headers: Add the authorization header
    * Access-Control-Allow-Methods: Add GET as an allowed method
  
This should allow you to access the API from the browser using CORS with something like:
```
let response = await fetch("https://YOURAPI.execute-api.us-west-2.amazonaws.com", {
  method: 'GET',
  headers: {
  'Authorization': 'Bearer ' + token.accessToken
  },
});

var data = await response.json();
```

HUZZAH!

This vastly simplified API Gateway v2 is really a gamechanger for us - the previous API Gateway worked well, but it was WAYYYY too complicated. And the ability to do JWT auth easily like this - beautiful. Well done AWS.
