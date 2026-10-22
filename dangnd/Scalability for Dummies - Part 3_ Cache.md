---
title: "Scalability for Dummies - Part 3: Cache"
source: "https://web.archive.org/web/20230126233752/https://www.lecloud.net/post/9246290032/scalability-for-dummies-part-3-cache"
author:
published: 2011-08-22
created: 2026-10-08
description: "After following Part 2 of this series, you now have a scalable database solution. You have no fear of storing terabytes anymore and the world is looking fine. But just for you. Your users still have..."
tags:
  - "clippings"
---
After following [Part 2](https://web.archive.org/web/20230126233752/http://www.lecloud.net/post/7994751381/scalability-for-dummies-part-2-database) of this series, you now have a scalable database solution. You have no fear of storing terabytes anymore and the world is looking fine. But just for you. Your users still have to suffer slow page requests when a lot of data is fetched from the database. The solution is the implementation of a cache.

With “cache” I always mean in-memory caches like [Memcached](https://web.archive.org/web/20230126233752/https://href.li/?http://memcached.org/) or [Redis](https://web.archive.org/web/20230126233752/https://href.li/?http://redis.io/). Please **never do file-based caching**, it makes cloning and auto-scaling of your servers just a pain.  
  
But back to in-memory caches. A cache is a simple key-value store and it should reside as a buffering layer between your application and your data storage. Whenever your application has to read data it should at first try to retrieve the data from your cache. Only if it’s not in the cache should it then try to get the data from the main data source. Why should you do that? Because **a cache is lightning-fast**. It holds every dataset in RAM and requests are handled as fast as technically possible. For example, Redis can do several hundreds of thousands of read operations per second when being hosted on a standard server. Also writes, especially increments, are very, very fast. Try that with a database!  
  
There are 2 patterns of caching your data. An old one and a new one:  
**  
#1 - Cached Database Queries  
**That’s still the most commonly used caching pattern. Whenever you do a query to your database, you store the result dataset in cache. A hashed version of your query is the cache key. The next time you run the query, you first check if it is already in the cache. The next time you run the query, you check at first the cache if there is already a result. This pattern has several issues. The main issue is the expiration. It is hard to delete a cached result when you cache a complex query (who has not?). When one piece of data changes (for example a table cell) you need to delete all cached queries who may include that table cell. You get the point?

**#2 - Cached Objects**  
That’s my strong recommendation and I always prefer this pattern. In general, see your data as an object like you already do in your code (classes, instances, etc.). Let your class assemble a dataset from your database and then store the complete instance of the class or the assembed dataset in the cache. Sounds theoretical, I know, but just look how you normally code. You have, for example, a class called “Product” which has a property called “data”. It is an array containing prices, texts, pictures, and customer reviews of your product. The property “data” is filled by several methods in the class doing several database requests which are hard to cache, since many things relate to each other. Now, do the following: when your class has finished the “assembling” of the data array, directly store the data array, or better yet the complete instance of the class, in the cache! This allows you to easily get rid of the object whenever something did change and makes the overall operation of your code faster and more logical.  
  
And the best part: it makes asynchronous processing possible! Just imagine an army of worker servers who assemble your objects for you! The application just consumes the latest cached object and nearly never touches the databases anymore!  
  
Some ideas of objects to cache:

- user sessions (never use the database!)
- fully rendered blog articles
- activity streams
- user<->friend relationships

As you maybe already realized, I am a huge fan of caching. It is easy to understand, very simple to implement and the result is always breathtaking. In general, I am more a friend of Redis than Memcached, because I love the extra database-features of Redis like persistence and the built-in data structures like lists and sets. With Redis and a clever key’ing there may be a chance that you even can get completly rid of a database. But if you just need to cache, take Memcached, because it scales like a charm.  
  
Happy caching!  
  
The already released parts of the series “ [Scalability for Dummies](https://web.archive.org/web/20230126233752/https://www.lecloud.net/tagged/scalability) ” you can find [here](https://web.archive.org/web/20230126233752/https://www.lecloud.net/tagged/scalability).

Filed under [scalability](https://web.archive.org/web/20230126233752/https://www.lecloud.net/tagged/scalability)