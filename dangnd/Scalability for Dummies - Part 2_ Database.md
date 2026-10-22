---
title: "Scalability for Dummies - Part 2: Database"
source: "https://web.archive.org/web/20220602114024/https://www.lecloud.net/post/7994751381/scalability-for-dummies-part-2-database"
author:
published: 2011-07-24
created: 2026-10-08
description: "After following Part 1 of this series, your servers can now horizontally scale and you can already serve thousands of concurrent requests. But somewhere down the road your application gets slower and..."
tags:
  - "clippings"
---
After following [Part 1](https://web.archive.org/web/20220602114024/https://www.lecloud.net/post/7295452622/scalability-for-dummies-part-1-clones) of this series, your servers can now horizontally scale and you can already serve thousands of concurrent requests. But somewhere down the road your application gets slower and slower and finally breaks down. The reason: your database. It’s MySQL, isn’t it?

Now the required changes are more radical than just adding more cloned servers and may even require some boldness. In the end, you can choose from 2 paths:

**Path #1** is to stick with MySQL and keep the “beast” running. Hire a database administrator (DBA,) tell him to do master-slave replication (read from slaves, write to master) and upgrade your master server by adding RAM, RAM and more RAM. In some months, your DBA will come up with words like “sharding”, “denormalization” and “SQL tuning” and will look worried about the necessary overtime during the next weeks. At that point every new action to keep your database running will be more expensive and time consuming than the previous one. You might have been better off if you had chosen Path #2 while your dataset was still small and easy to migrate.

**Path #2** means to denormalize right from the beginning and include no more Joins in any database query. You can stay with MySQL, and use it like a NoSQL database, or you can switch to a better and easier to scale NoSQL database like MongoDB or CouchDB. Joins will now need to be done in your application code. The sooner you do this step the less code you will have to change in the future. But even if you successfully switch to the latest and greatest NoSQL database and let your app do the dataset-joins, soon your database requests will again be slower and slower. You will need to introduce a cache.

The other parts of the series “ [Scalability for Dummies](https://web.archive.org/web/20220602114024/https://www.lecloud.net/tagged/scalability) ” you can (soon) find [here](https://web.archive.org/web/20220602114024/https://www.lecloud.net/tagged/scalability).

Filed under [scalability](https://web.archive.org/web/20220602114024/https://www.lecloud.net/tagged/scalability)