---
title: "Heatmaps By Hexos"
date: 2022-01-10T09:47:18Z
draft: false
summary: Running between 2019 and 2022, Hexos was a UK provider of realistic driving and public transport based heatmaps/travel-time maps for data visualisation. 
---

![Heatmap](/images/heatmaps-by-hexos/heatmap-cover.png)

## What Is Hexos? 

Hexos is the name of my self-employed business, through which I develop web applications, mobile applications as well as provide general IT solutions. 

## What Is 'Heatmaps By Hexos'?

Heatmaps by Hexos is one of the web/cloud applications I developed and ran between 2019 and 2022. It was a successful business that sold heatmap images to schools and colleges across the UK. 

> Hexos has developed a service that can create heatmaps / travel time maps to visualise public transport and driving commute times travelling towards or from a central point within a given radius. Heatmaps can be generated for any time of day, any day of the week and any location in the United Kingdom.

## The Tech Stack

The tech stack was as follows:
* Express web application - Node.js
* AWS Lambda micro-services - Node.js
* AWS SQS (Simple Queue Service)
* AWS DynamoDB (NoSQL Object Database)
* Bootstrap Front-End
* Leaflet.js Map Provider
* Google Maps Directions API

## An Example Of The End Product

The main product that Heatmaps By Hexos generated was a selection of high definition images (also viewable as a interactive online version) - showing the travel times for the given configuration (postcode, day of the week, time of the day, transit type etc). 

Here is an example:

![Example heatmap](/images/heatmaps-by-hexos/heatmap-2048.png)

## A Guide Around The Web Application

I'll try to show you what the web application looked like, and how customers went through the process of requesting a heatmap from us. 

### Login Page

Customers could login and sign up via our secure login and register pages. We also had forgot password functionality that would send out a reset password link with a 24 hour expiry. 

![Login page](/images/heatmaps-by-hexos/login.png)

### Dashboard

On the dashboard, the user had access to all the heatmaps they had previously or actively requested. Each heatmap was described by the postcode it represented. You could track the status of a heatmap if it was currently in the process of being generated, and once generated, view the interactive heatmap or download the images. 

![Dashboard](/images/heatmaps-by-hexos/heatmap-list.png)

### Request Heatmap

On this page, the customer used pre-purchased credits to request a heatmap, by defining multiple settings:

The size of the heatmap defined the maximum search radius - that is, how far the algorithm would search for possible routes. 
![Request page](/images/heatmaps-by-hexos/request-1.png)

The postcode represented the centre of the heatmap, either where all journeys started or finished. The map would update to place a pin at the postcode they had picked. 
![Request page](/images/heatmaps-by-hexos/request-2.png)

The journeys could either be calculated by using routes taken by car, or routes taken public transport. 

They then had to decide whether the heatmap was inbound or outbound. Inbound heatmaps would consider the latest time required to leave at a location to arrive at the centre by a given time. Outbound heatmaps would consider the earliest time you could arrive at a point, given you left the centre at a given time. 
![Request page](/images/heatmaps-by-hexos/request-3.png)

The arrival or departure time would need to be set dependant on if they had picked an inbound or outbound heatmap. The day of the week then had to be picked, so that the algorithm could look at relevant routes. For example traffic may be worse in the week, or public transport not as frequent on a weekend. 
![Request page](/images/heatmaps-by-hexos/request-4.png)

The user then had to submit their request - at which point the heatmap would begin processing on the AWS cloud system. 
![Request page](/images/heatmaps-by-hexos/request-5.png)

When the user requested a heatmap they would receive an email:
![Email](/images/heatmaps-by-hexos/request-received-email.png)

And another email when the heatmap was ready.
![Email](/images/heatmaps-by-hexos/heatmap-ready-email.png)

### Interactive Heatmap Viewer

The interactive heatmap viewer was very similar to the downloadable images, except it let you zoom in much further, click on any cell to get specific route information, and change the maximum time filter. 

![Heatmap viewer](/images/heatmaps-by-hexos/heatmap-viewer.png)
![Heatmap viewer](/images/heatmaps-by-hexos/heatmap-viewer-info.png)
![Heatmap viewer](/images/heatmaps-by-hexos/heatmap-viewer-settings.png)

### Account Settings

There was also an account settings page where you could view all the information we stored, as well as change useful settings such as turning email updates off. 

![Settings](/images/heatmaps-by-hexos/settings-1.png)
![Settings](/images/heatmaps-by-hexos/settings-2.png)

### Purchase Credits

Customers had to buy credits through the online dashboard that would then be credited to their account. They could then use these credits against eligible heatmap requests.

For example a `city` heatmap would cost 2 credits. 

Payments were processed through `Stripe`. 

![Settings](/images/heatmaps-by-hexos/purchase-credits.png)

### Heatmap Download

The user could then download a zip folder of images, in 4 definitions ranging from `512x512` to `3072x3072`. The images were then broken up into 5 maximum time filters, from 1 to 3 hours. 

We also supplied the user with a GeoJSON file with all the heatmap data, so they could make their own images. 

## What Has Happened To 'Heatmaps By Hexos' Now?

Heatmaps by Hexos was a successful business that sold heatmap images to schools and colleges across the UK. They used these images to find new recruitment areas for students to come from that they hadn't previously considered. 

In 2022 I made the tough decision to shut this service down in order to focus on my professional development as a software engineer in industry. Other factors included the increasing time required to maintain the online service, especially with the speed at which frameworks such as Node.js are moving. 

I learnt invaluable lessons whilst building this product as well as building my network of connections, and I am sure that this is not the last we will see of Hexos!

## Other Resources & Information

These items are considered as archived - but give some extra details about specific parts of Heatmaps by Hexos, and therefore may be of interest. These items were mainly used as marketing material. 

### Sheffield Digital Interview

[![Digital Showcase: Dom Barter from Hexos](http://img.youtube.com/vi/7uY_fDDKOMs/0.jpg)](http://www.youtube.com/watch?v=7uY_fDDKOMs "Digital Showcase: Dom Barter from Hexos")

### Heatmap Handbook

[The Heatmap Handbook](/documents/the-heatmap-handbook.pdf)

### Service Overview

[Service Overview](/documents/service-overview.pdf)

### Example Heatmap Download

[heatmap.zip](/documents/heatmap.zip)