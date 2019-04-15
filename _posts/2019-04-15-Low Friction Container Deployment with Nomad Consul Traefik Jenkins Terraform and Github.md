---
title:  "Low Friction Container Deployment with Nomad, Consul, Traefik, Jenkins, Terraform, and Github"
tags: [github, terraform, consul, nomad, jenkins, traefik]
---

Like many, I've been looking for a way to deploy containers in our environment securely and efficiently with a minimal amount of fuss. After taking a hard look at Kubernetes, I came to the realization it might be a little too complex to roll out in our environment, so I started looking at Consul + Nomad. I'm currently using Terraform and Vault heavily in our environment, and I'm pretty comfortable with the HashiCorp way of doing things, so this seemed like a good option.

Pretty quickly, we were able to get something working, but without a load balancer doing the translation between the randomized ports Docker + Nomad gives you, fitting it into our environment would be a challenge. FabioLB seemed pretty popular, but I found the documentation really lacking in some areas, so I concentrated my efforts on Traefik.

### Getting Traefik Going

Since getting the load balancing right was important, this is where I spent the bulk of my time.

Things we wanted from the solution:
* HTTPS for web services with no manual steps required (no manual cert provisioning or mapping)
* "Dynamic" DNS (no manual DNS changes required to put a solution into PROD).

This is what we came up with.
[Diagram]({{ site.url }}/assets/ConsulNomadTraefikJenkins.PNG)
