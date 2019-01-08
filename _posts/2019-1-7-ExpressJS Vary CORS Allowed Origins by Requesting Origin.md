---
title:  "ExpressJS Vary CORS Allowed Origins by Requesting Origin"
tags: [aws, webdev, nodejs]
---

We use Cloudfront Signed Cookies (https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-signed-cookies.html) to provide secure access to JSON documents stored in S3. The cookies are provisioned by a small webservice that was initially rolled out for a single web application (a great topic for another blog post), but was found to be so useful we used it for a bunch of apps. Unfortunately, we quickly ran into the "cross domain cookie" issue, where the browser will silently drop cookies set via AJAX from a subdomain unless requested with withCredentials, which subsequently requires that the origin not be set to "*". Here's how we solved that with our NodeJS microservice.

### Initial Code - Using Express Application-level Middleware to Set our Headers
```
app.use(function(req, res, next) {
  res.header("Access-Control-Allow-Origin", "*");
  res.header("Access-Control-Allow-Headers", "Origin, X-Requested-With, Content-Type, Accept, x-api-key");
  res.header("Cache-Control", "no-cache");
  next();
});
```

### Updated Code - Varying the Access-Control-Allow-Origin Depending on the Requesting Origin
```
app.use(function(req, res, next) {
  //use req.headers.origin to set a dynamic origin - respond with the correct allow-origin depending on the origin making the call
  //this is necessary for cross-domain cookies (which requires withCredentials, which in turn requires we not use * as the allowed origin)
  res.header("Access-Control-Allow-Origin", req.headers.origin);
  res.header("Access-Control-Allow-Headers", "Origin, X-Requested-With, Content-Type, Accept, x-api-key");
  res.header("Access-Control-Allow-Credentials", true);
  res.header("Cache-Control", "no-cache");
  next();
});
```

### Calling the Code to Set the Cloudfront Cookies with New withCredentials Login
```
$(document).ready(function() {
     var hostname = 'https://mysite.mydomain.com/login';
     $.ajax({
      url: hostname,
      xhrFields: {
         withCredentials: true
      },
      success: function(data) {
             console.log('set cookies!');
      },
      error: function(jqXHR, textStatus, errorThrown) {
             alert('error ' + textStatus + " " + errorThrown);
      }
     });
 });
```
