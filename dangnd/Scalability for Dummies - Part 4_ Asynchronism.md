---
title: "Scalability for Dummies - Part 4: Asynchronism"
source: "https://web.archive.org/web/20220926171507/https://www.lecloud.net/post/9699762917/scalability-for-dummies-part-4-asynchronism"
author:
published: 2011-09-02
created: 2026-10-08
description: "In general, there are two ways / paradigms asynchronism can be done. Async #1Let’s stay in the former bakery picture. The first way of async processing is the “bake the breads at night and sell..."
tags:
  - "clippings"
---
This 4th part of the [series](https://web.archive.org/web/20220926171507/http://www.lecloud.net/tagged/scalability) starts with a picture: please imagine that you want to buy bread at your favorite bakery. So you go into the bakery, ask for a loaf of bread, but there is no bread there! Instead, you are asked to come back in 2 hours when your ordered bread is ready. That’s annoying, isn’t it?  
  
To avoid such a “please wait a while” - situation, asynchronism needs to be done. And what’s good for a bakery, is maybe also good for your web service or web app.

In general, there are two ways / paradigms asynchronism can be done.  
**  
Async #1  
**Let’s stay in the former bakery picture. The first way of async processing is the “bake the breads at night and sell them in the morning” way. No waiting time at the cash register and a happy customer. Referring to a web app this means doing the time-consuming work in advance and serving the finished work with a low request time.  
  
Very often this paradigm is used to turn dynamic content into static content. Pages of a website, maybe built with a massive framework or CMS, are pre-rendered and locally stored as static HTML files on every change. Often these computing tasks are done on a regular basis, maybe by a script which is called every hour by a cronjob. This pre-computing of overall general data can extremely improve websites and web apps and makes them very scalable and performant. Just imagine the scalability of your website if the script would upload these pre-rendered HTML pages to AWS S3 or Cloudfront or another Content Delivery Network! Your website would be super responsive and could handle millions of visitors per hour!  
**  
Async #2  
**Back to the bakery. Unfortunately, sometimes customers has special requests like a birthday cake with “Happy Birthday, Steve!” on it. The bakery can not foresee these kind of customer wishes, so it must start the task when the customer is in the bakery and tell him to come back at the next day. Refering to a web service that means to handle tasks asynchronously.  
  
Here is a typical workflow:  
  
A user comes to your website and starts a very computing intensive task which would take several minutes to finish. So the frontend of your website sends a job onto a job queue and immediately signals back to the user: your job is in work, please continue to the browse the page. The job queue is constantly checked by a bunch of workers for new jobs. If there is a new job then the worker does the job and after some minutes sends a signal that the job was done. The frontend, which constantly checks for new “job is done” - signals, sees that the job was done and informs the user about it. I know, that was a very simplified example.  
  
If you now want to dive more into the details and actual technical design, I recommend you take a look at the first 3 tutorials on the [RabbitMQ website](https://web.archive.org/web/20220926171507/https://href.li/?http://www.rabbitmq.com/). RabbitMQ is one of many systems which help to implement async processing. You could also use [ActiveMQ](https://web.archive.org/web/20220926171507/https://href.li/?http://activemq.apache.org/) or a simple [Redis list](https://web.archive.org/web/20220926171507/https://href.li/?http://redis.io/topics/data-types). The basic idea is to have a queue of tasks or jobs that a worker can process. Asynchronism seems complicated, but it is definitely worth your time to learn about it and implement it yourself. Backends become nearly infinitely scalable and frontends become snappy which is good for the overall user experience.  
  
If you do something time-consuming, try to do it always asynchronously.  
  
The already released parts of the series [“Scalability for Dummies”](https://web.archive.org/web/20220926171507/http://www.lecloud.net/tagged/scalability) you can find here.

Filed under [scalability](https://web.archive.org/web/20220926171507/https://www.lecloud.net/tagged/scalability)