Welcome to DataStax Essentials Day!
===================
![icon](http://i.imgur.com/FoIOBlt.png)

In this session, you'll learn all about DataStax Enterprise. It's a mix between presentation and hands-on. This is **obviously** your reference for the hands-on content. Feel free to bookmark this page for future reference! 

----------


Logistics
-------------

We have an 8 node cluster for you to play with! The cluster is currently running in both **search** and **analytics** mode so you can take advantage of both Spark and Solr on your Cassandra data. 

```
// To SSH into the cluster:
ssh datastax@ipaddress 
// You can login to any of the nodes from x.x.x.1 to x.x.x.8 
password: XXXXX
```

#### UI's you'll want to play around with
> - OpsCenter: www.linktoopscenter.com
> - Spark: www.linktospark.com
> - Solr: www.linktosolr.co

#### Connecting to the cluster from DevCenter
- Simply add a new connection
- Enter a name for your connection
- Enter any of the ip's from above as a contact host
- Profit. 

![DevCenter How To](http://i.imgur.com/8zwzmDj.png)


----------


DSE Cassandra 
-------------------

Cassandra is the brains of DSE. It's an awesome storage engine that handles replication, availability, structuring, and of course, storing the data at lightning speeds. It's important to get yourself acquainted with the Cassandra to fully utilize the power of the DSE Stack. 

**Commands you'll want to know:**
```
nodetool  //Cassandra's main utility tool
```
**Examples:**
```
nodetool status  //shows current status of the cluster 
nodetool tpstats //shows thread pool status - critical for ops
```

```
dsetool //main utility tool for DSE
```
**Examples:**
```
dsetool status //shows current status of cluster, including DSE features

dsetool create_core //will create a Solr schema on Cassandra data for Search
```

**The main log you'll be taking a look at:**
```/var/log/cassandra/system.log```


####Querying the database 

In addition to DevCenter, you can also use **CQLSH** as an interactive command line too for query data in Cassandra. 

```cqlsh 127.0.0.1``` 
> Make sure to replace 127.0.0.1 with the IP of the respective node 

Let's make our first Cassandra Keyspace! 
>Hint: You can use DevCenter if you'd like to use something that's easy on the eyes. 

```CREATE KEYSPACE <Enter your name> WITH replication = {'class': 'SimpleStrategy', 'replication_factor': 3 };```

And just like that, any data within any table you create under your keyspace will automatically be replicated 3 times. Let's keep going and create ourselves a table. You can follow my example or be a rebel and roll your own. 

```
CREATE TABLE <yourkeyspace>.sales (
	name text,
	time int,
	item text,
	price double,
	PRIMARY KEY (name, time)
) WITH CLUSTERING ORDER BY ( time DESC );

```
>Yup. This table is very simple but don't worry, we'll play with some more interesting tables in just a minute.

Let's get some data into your table! Do this a few times with different values. 

```INSERT INTO <yourkeyspace>.sales (name, time, item, price) VALUES ('marc', 20150205, 'Apple Watch', 299.00);```

And to retrieve it:

```SELECT * FROM <keyspace>.sales where name='marc' AND time >=20150205 ;
```
>See what I did there? You can do range scans on clustering keys! Give it a try.

#### Let's play with consistency!

Consistency in Cassandra refers to the number of acknowledgements replica nodes need to send to the coordinator for an operation to be successful. This is something that developers need to spend time planning. In most cases, developers find Cassandra's replication fast enough to warrant lower consistency for better SLA's. For cases where stronger consistency is required, a developer can trade latency for a higher consistency level. Let's give it a shot. 

>During this exercise, I'll be taking down nodes so you can see the CAP theorem in action. We'll be using CQLSH for this one. 

**In CQLSH**:

```tracing on```
```consistency all```

>Any query will now be traced. **Consistency** of all means all 3 replicas need to respond to a given request (read OR write) to be successful. Let's do a **SELECT** statement to see the effects.

```SELECT * FROM <yourkeyspace>.sales where name='<enter name>';```

How did we do? 

**Let's compare a lower consistency level:**
```consistency local_quorum```
>Quorum means majority: RF/2 + 1. In our case, 3/2 = 1 + 1 = 2. At least 2 nodes need to acknowledge the request. 

Let's try the **SELECT** statement again. Any changes in latency? 
>Keep in mind that our dataset is so small, it's sitting in memory on all nodes. With larger datasets that spill to disk, the latency cost become much more drastic. 

Keep trying different combinations of **consistency level** and types of select statements. Remove the partition key from your query and look at what happens in the trace. It's interesting to see what's happening in the trace but keep in mind that these are extremely important concepts you, as the developer, need to grasp. 

For more detailed classed on data modeling, consistency, and Cassandra 101, check out the free classes at the [DataStax Academy](www.academy.datastax.com) website. 

----------


DSE Search
-------------
DSE Search is awesome. You can configure which columns of which Cassandra tables you'd like indexed in **lucene** format to make extended searches more efficient while enabling features such as text search and geospatial search. 

Let's start off by indexing the tables we've already made. Here's where the dsetool really comes in handy:

```dsetool create_core <yourkeyspace>.sales generateResources=true reindex=true```

>If you've ever created your own Solr cluster, you know you need to create the core and upload a schema and config.xml. That **generateResources** tag does that for you. For production use, you'll want to take the resources and edit them to your needs but it does save you a few steps. 

This by default will map Cassandra types to Solr types for you. Anyone familiar with Solr knows that there's a REST API for querying data. In DSE Search, we embed that into CQL so you can take advantage of all the goodness CQL brings. Let's give it a shot. 

```
SELECT * FROM <keyspace>.<table> WHERE solr_query=‘{“q”:”column:*”}’;

SELECT * FROM <keyspace>.sales WHERE solr_query='{"q":”name:marc", "fq":”item:*pple*", "sort":”product:asc"}’; 
```
> For your reference, [here's the doc](http://docs.datastax.com/en/datastax_enterprise/4.8/datastax_enterprise/srch/srchCql.html?scroll=srchCQL__srchSolrTokenExp) that shows some of things you can do

OK! Time to work with some more interesting data. Meet Amazon book sales data:
>Note: This data is already in the DB, if you want to try it at home, [CLICK ME](https://github.com/Marcinthecloud/Solr-Amazon-Book-Demo). 

Click stream data:
```
CREATE TABLE amazon.clicks (
    asin text,
    seq timeuuid,
    user uuid,
    area_code text,
    city text,
    country text,
    ip text,
    loc_id text,
    location text,
    location_0_coordinate double,
    location_1_coordinate double,
    metro_code text,
    postal_code text,
    region text,
    solr_query text,
    PRIMARY KEY (asin, seq, user)
) WITH CLUSTERING ORDER BY (seq DESC, user ASC);
```
And book metadata: 

```
CREATE TABLE amazon.metadata (
    asin text PRIMARY KEY,
    also_bought set<text>,
    buy_after_viewing set<text>,
    categories set<text>,
    imurl text,
    price double,
    solr_query text,
    title text
);
```

So what are things you can do? 
>Filter queries: These are awesome because the result set gets cached in memory. 
```
SELECT * FROM amazon.metadata WHERE solr_query='{"q":"title:Noir~", "fq":"categories:Books", "sort":"title asc"}' limit 10; 
```
> Faceting: Get counts of fields 
```
SELECT * FROM amazon.metadata WHERE solr_query='{"q":"title:Noir~", "facet":{"field":"categories"}}' limit 10; 
```
> Geospatial Searches: Supports box and radius
```
SELECT * FROM amazon.clicks WHERE solr_query='{"q":"asin:*", "fq":"+{!geofilt pt=\"37.7484,-122.4156\" sfield=location d=1}"}' limit 10; 
```
> Joins: Not your relational joins. These queries 'borrow' indexes from other tables to add filter logic. These are fast! 
```
SELECT * FROM amazon.metadata WHERE solr_query='{"q":"*:*", "fq":"{!join from=asin to=asin force=true fromIndex=amazon.clicks}area_code:415"}' limit 5; 
```
> Fun all in one. 
```
SELECT * FROM amazon.metadata WHERE solr_query='{"q":"*:*", "facet":{"field":"categories"}, "fq":"{!join from=asin to=asin force=true fromIndex=amazon.clicks}area_code:415"}' limit 5;
```
Want to see a really cool example of a live DSE Search app? Check out [KillrVideo](http://www.killrvideo.com/) and its [Git](https://github.com/luketillman/killrvideo-csharp) to see it in action. 

----------


DSE Analytics (Spark)
--------------------




#####For at home use, you need:
* Python 2.7+
* [DataStax Python Driver](https://github.com/datastax/python-driver) 
* [DataStax Enterprise 4.8.3 or greater](https://www.datastax.com/downloads) 