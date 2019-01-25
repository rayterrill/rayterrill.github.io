---
title:  "Using AzureAD ADALJS PassPortJS and Passport-Azure-AD to Protect a NodeJS API"
tags: [nodejs, ad, webdev]
---

We have a few NodeJS APIs we're looking to protect with AzureAD, our chosen identity solution. We ended up picking the ADAL JS library from Microsoft for the client-side of our app, and PassportJS + passport-azure-ad for the back end authentication.

### Front End ADAL Config to Use MS Graph

The front end was pretty easy to get working in a proof-of-concept. I ended up taking the Microsoft SPA example and simplifying it some, publishing that work here: (https://github.com/rayterrill/AzureADADALv1Example)[https://github.com/rayterrill/AzureADADALv1Example].

Where things became tricky for me was trying to understand how we needed to request tokens to access APIs from our client-side app. It appears through reading the documentation and some trial and error that we need to set the resource in the authContext.acquireToken function to the service we need the token for, which makes sense.

So for a call to MSGraph, here's the way that function looks:
```
//acquire token for ms graph. the service we're acquiring a token for should be the same service we call in the ajax request below
authContext.acquireToken('https://graph.microsoft.com', (error, token) => {
   // Handle ADAL Error
   if (error || !token) {
      console.log('ADAL Error Occurred: ' + error);
      return;
   }
                    
   //at this point we have the token for the graph API in 'token' and can do whatever we need with it
});
```

That token can now be used to make a call into the MS Graph API, like the following:
```
$.ajax({
   type: "GET",
   url: "https://graph.microsoft.com/v1.0/me/mailboxSettings/workingHours",
   headers: {
      'Authorization': 'Bearer ' + token, //uses the token we got above
   }
}).done((data) => {
   console.log(data);
});
```

I ended up building a pretty simple VueJS app (https://github.com/rayterrill/MSGraphVueJSDashboard)[(https://github.com/rayterrill/MSGraphVueJSDashboard] to illustrate how this works in full practice.

### Calling into Our Own NodeJS API

But what if we want to call into one of our own APIs, vs MS Graph? In this case, we have an app that has a back-end API that's a part of the same app, but is implemented in NodeJS and running in AWS. Here's how we accomplished this (this took the most work - I had a hell of a time finding a full-functional example).

First, we define our configuration in our config.js file (logging enabled to help troubleshoot):
```
const tenantName    = 'OUR_TENANT_NAME';
const clientID      = 'CLIENT_ID_OF_OUR_APPLICATION_IN_AZUREAD';
const serverPort    = 3000;

module.exports.serverPort = serverPort;

module.exports.credentials = {
  identityMetadata: `https://login.microsoftonline.com/${tenantName}.onmicrosoft.com/.well-known/openid-configuration`, 
  clientID: clientID,
  loggingLevel: 'info'
};
```

After that we define our actual API, with an /api route protected by AzureAD:
```
// index.js
const passport = require('passport');
const BearerStrategy = require('passport-azure-ad').BearerStrategy;
const config = require('./config');
const serverPort = process.env.PORT || config.serverPort;
var cors = require('cors')

const express = require('express')
const app = express()

//build out our azuread bearer strategy
const authenticationStrategy = new BearerStrategy(config.credentials, (token, done) => {
  let currentUser = null;

  console.log('currentUser: ' + token.upn);

  return done(null, token.upn, token);
});

//tell passport to use our new authentication strategy
passport.use(authenticationStrategy);

app.use(cors())

app.use(passport.initialize());
app.use(passport.session());

//unprotected endpoint
app.get('/', function (req, res) {
  res.send('Hello World!')
})

//endpoint protected by azuread bearar authentication
app.get('/api', passport.authenticate('oauth-bearer', { session: false }), (req, res) => {
  res.json({ message: 'response from protected API endpoint' });
});

app.listen(serverPort, () => console.log(`Example app listening on port ${serverPort}!`))
```

And then to call into our API, we need to request a token for this API. NOTE: Because this is a back-end API that is a part of our client-side APP, we need to request a token for the client_id of our client-side app (this tripped me up for hours):
```
authContext.acquireToken('CLIENT_ID_OF_OUR_APPLICATION_IN_AZUREAD', (error, token) => {
   // Handle ADAL Error
   if (error || !token) {
      console.log('ADAL Error Occurred: ' + error);
      return;
   }
                    
   //at this point we have the token for our private API in 'token' and can do whatever we need with it
});
```

So now let's call into our API:
```
$.ajax({
   type: "GET",
   url: "http://localhost:3000/api",
   headers: {
      'Authorization': 'Bearer ' + token,
   }
}).done((data) => {
   console.log(data)
}).fail(() => {
   console.log('Error hitting our private API');
});
```

I published a functional example of the Express/NodeJS API here: (https://github.com/rayterrill/ExpressPassportAzureADAPIExample)[https://github.com/rayterrill/ExpressPassportAzureADAPIExample]
