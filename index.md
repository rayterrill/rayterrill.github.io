---
title: "Home"
---

### Latest Blog Posts

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul>

### Manual

* [API Gateway with Multiple Deployments](APIGatewayMultipleDeployments.md)
* [Getting Github AutoDeployments Working with AWS CodeDeploy](GettingGithubAutoDeploymentsWorkingWithAWSCodeDeployAPIGatewayAndLambda.md)
