---
layout: post
title: Portfolio
---

Here is an overview of some of the projects I've worked on. Thanks for stopping by!

E-RAS - Electronic Record of Authorized Signatures
--------------------------------------------------

This app is a multi-tenant account repository that stores information about financial accounts. Users can only see/edit the accounts they have access to. The app was built in C# 4 using the ASP.NET MVC 3 framework. I used Lucene.NET to full-text index all searchable data, and RabbitMQ to send messages from the web app to and from the Lucene index.

<iframe width="560" height="315" src="http://www.youtube.com/embed/hggiplu92Xk?rel=0&cc_load_policy=1" frameborder="0" allowfullscreen></iframe>

- - -

"Starter Site" NuGet Package for ASP.NET MVC
--------------------------------------------------

Information coming soon!

- - -

GTIS - Green Transit Information System
----------------------------------------

This was an extremely fun and challenging project to work on involving a bunch of different technologies. The chief requirement was collecting information on which groups of the campus community were using our campus bus service. For example, did staff use our bus service, or was it mostly students? Graduate or undergraduate? Where were they boarding the bus? To accomplish this, we developed a C# / WPF application running on a laptop that would be installed in the bus. This application used a card reader to obtain information from an individual's University ID card, and a GPS adapter to determine the location.

A second requirement, to provide a service to the campus community, was to display information about the route such as the next and upcoming stops. Our app also announced when the bus was arriving at a stop, and what the next stop was.

Here's a brief video showing a prototype of the app:

<iframe width="420" height="315" src="http://www.youtube.com/embed/0fw5ha1hYEM?rel=0&cc_load_policy=1" frameborder="0" allowfullscreen></iframe>

Unfortunately, the app never went live due to changing goals and a reallocation of resources. This project became what is now the [SBU Smart Transit System](http://smarttransit.cewit.stonybrook.edu/smarttransit/). Even though this app didn't go live, building it was extremely rewarding and I learned a lot about threading, WPF, unit testing, and coding in general.