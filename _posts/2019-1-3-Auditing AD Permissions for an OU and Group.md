---
title:  "Auditing AD Permissions for an OU and Group"
tags: [ad, powershell, windows]
---

Like many people, we have an AD implementation that has undergone a number of refactorings over the years, resulting in permissions delegated and applied at various levels. Troubleshooting things like what permissions a user or group has access to always seems like a pain - the AD GUI it not great at showing a quick summary of the discrete permissions for a particular account.

I put together a quick PowerShell script that takes an OU and a user/group as parameters, and shows the permissions applied to that user/group on that particular OU. Hopefully someone else will find it useful!

<script src="https://gist.github.com/rayterrill/739dfe730328a98a05d1467dc53c6589.js"></script>
