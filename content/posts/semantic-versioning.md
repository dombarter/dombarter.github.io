---
title: "Semantic Versioning"
date: 2022-01-09T16:22:18Z
draft: false
summary: "How to correctly version things using the x.y.z format"
---

## What does `x.y.z` stand for?

In a given version number, `x.y.z` stands for `major.minor.patch`. 

E.g `1.2.4` means:
* Major release: 1
* Minor release: 2
* Patch release: 4

## Major Versions

Major versions should be incremented when you introduce huge change to a project, and specifically when you are making something no longer backwards compatible. 

## Minor Versions

Minor versions should be incremented when you introduce a new feature or a small to medium change. The product remains mostly the same but you have introduced something new.

## Patch Versions

Patch versions should be incremented when you fix a bug, or tweak a small setting. A very small change. 

## Incrementing Version Numbers

When incrementing versions, major takes priority over minor and minor takes priority over patch. This means when you increment a minor version, the patch version gets reset to 0, and when you increment a major version, the minor version is reset to 0. For example:
* `1.0.0`
* `1.0.1`
* `1.1.0`
* `1.2.0`
* `2.0.0`