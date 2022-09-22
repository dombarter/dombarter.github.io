---
title: "How to correctly name your Azure resources"
date: 2022-09-22T15:58:23+01:00
draft: false
summary: "Every wondered if there is a better way of naming your Azure resources than my-database and my-app? Well, there is."
---

Azure resources should be named like so, we'll break down each section shortly.

![Resource naming](/images/resource-naming.png)
_From https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/resource-naming_

## Resource Type

The resource type is what type of infrastructure you are running. It could be a database, virtual machine or application insights etc. It's recommended practice to use an abbreviated name such as `ase`, `vm` or `ai`. You can find the full list here: https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/resource-abbreviations

## Workload/Application

The workload/application is just a name that has some contextual meaning to you. Perhaps you're building a shop web application, then you could use `shop` as your application.

## Environment

The environment could be one of the following:

- Development - `dev`
- Staging - `stag`
- Production - `prod`

## Azure Region

The Azure region is an abbreviation of where the resource is geographically located. For example UK South would be `uksouth`. You can find the full list here: https://azuretracks.com/2021/04/current-azure-region-names-reference/

## Instance

And finally, you may have more than one instance of a resource running - such as a virtual machine; just increment the number to denote this.

# Resources

- https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/resource-naming
- https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/resource-abbreviations
- https://azuretracks.com/2021/04/current-azure-region-names-reference/
