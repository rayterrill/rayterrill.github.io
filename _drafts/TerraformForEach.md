---
title:  "Useful Terraform for_each Patterns and Data Structures"
tags: [terraform]
---

I find myself building the same data structures in Terraform over and over again. Documented some of the patterns I use the most so I can refer back to them. :)

### Build a Static Map of Things to Do Something With

This pattern is useful when I have a list of relatively static things I want to loop over.

Here's an example:
```
#variable definition in a module "widgets"
variable "widgets" {
  type = map(object({
    type = string
    content = string
    priority = number
  }))
  description = "a definition of some widgets"
  validation {
    condition = alltrue([
      for w in var.widgets : w.priority >= 200 && w.priority < 800
    ])
    error_message "priority must be between 200 and 800"
  }
  default = {}
}

#consumption of the variable
locals {
  widgets = tomap({
    "widget1" = {
      type = "small"
      content = "small widget"
      priority = 201
    }
  })
}

module "widgets" {
  source = "../modules/widgets"
  
  widgets = local.widgets
}
```

### Build a Dynamic Map of Things to Do Something With

This pattern is similar to the above pattern, but it's dynamically generating the map keys.

```
#variable definition in a module "widgets"
variable "widgets" {
  type = map(object({
    type = string
    content = string
    priority = number
  }))
  description = "a definition of some widgets"
  validation {
    condition = alltrue([
      for w in var.widgets : w.priority >= 200 && w.priority < 800
    ])
    error_message "priority must be between 200 and 800"
  }
  default = {}
}

#consumption of the variable
locals {
  widgets = ({
    for b in var.widgetsToCreate: b => {
      type = "test"
      content = "test widget"
      priority = 201
    }
  })
}

module "widgets" {
  source = "../modules/widgets"
  
  widgets = local.widgets
}
```
}

### Build a List of Objects from Multiple Lists

I use this one a lot - probably too much. It's just super useful. Taken directly from: https://developer.hashicorp.com/terraform/language/functions/flatten#flattening-nested-structures-for-for_each

```
locals {
  # flatten ensures that this local value is a flat list of objects, rather
  # than a list of lists of objects.
  network_subnets = flatten([
    for network_key, network in var.networks : [
      for subnet_key, subnet in network.subnets : {
        network_key = network_key
        subnet_key  = subnet_key
        network_id  = aws_vpc.example[network_key].id
        cidr_block  = subnet.cidr_block
      }
    ]
  ])
}

resource "aws_subnet" "example" {
  # local.network_subnets is a list, so we must now project it into a map
  # where each key is unique. We'll combine the network and subnet keys to
  # produce a single unique key per instance.
  for_each = {
    for subnet in local.network_subnets : "${subnet.network_key}.${subnet.subnet_key}" => subnet
  }

  vpc_id            = each.value.network_id
  availability_zone = each.value.subnet_key
  cidr_block        = each.value.cidr_block
}
```

### Take a JSON Structure and Turn it into Something You can for_each

This one builds on the one above a bit. It's a little more intense, but super useful if you want to allow team members who aren't familiar with Terraform manage data in a json file, but still let terraform manage the actual changes.

```
#groupmembers.json
{
    "Group1": [
        "user1@domain.com"
    ],
    "Group2": [
        "user1@domain.com",
        "user2@domain.com"
    ],
}

#manageGroups.tf
locals {
  groupmembers = jsondecode(file("${path.module}/groupmembers.json"))

  //terraform's for_each works on on a map or set of strings
  //this code generates a flattened list from the data contained in groupmembers.json - effectively providing a list where each entry contains the group and member to add to the group
  flattenedmembers = flatten([
    //iterate the keys in the groupmembers file
    for g in keys(local.groupmembers) : [
      //iterate the groupmembers in the key and flatten things out so its usable by for_each
      for m in local.groupmembers[g] : {
        key = format("%s_%s",g,m)
        group = g
        member = m
      }
    ]
  ])
}

module "groupmember" {
  source   = "../../../modules/assignment"

  //convert our list of tuples into a map so for_each works correctly
  for_each = {
    for i in local.flattenedmembers: i.key => i
  }

  user = each.value.member
  group = each.value.group
}
```
