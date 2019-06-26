# Study-Business-Analytics-II-Practice
HDFS, MapReduce, Apache Spark, PostgreSQL

---------------------------------------------------------------------------------------------------------------------------
> A relation means structure. Relational Database(SQL) is suitable for realtime crud(create/read/update/delete) operation while a big data stack(Hadoop) has a "create once/read many" type of file system, storing both structured/unstructured data without need of relations, consistency of format, etc. 

### Intro to MapReduce and Hadoop
Hadoop splits the data up and stores it across the collection of machines(a cluster) then processes the data in place **where it's actually stored(within the cluster)** rather than retrieving the data from a central server. Of course we can add more machines to the cluster as the amount of the data we storing grows(machines in the cluster is typically `mid-range servers`).
 - MapReduce is for processing
 - HDFS is for storing
> Hadoop_EcoSystem:
<img src="https://user-images.githubusercontent.com/31917400/44554202-f7566e80-a727-11e8-862c-a3a2d034549c.jpg" />

In addition to **Core Hadoop**(HDFS+MapReduce), other software has grown up around it to make Hadoop easier to use. For example, Writing MapReduce is not simple and this is where Hive or Pig come into play. In **Hive**, we can just write SQL-like statement, and Hive interpreter turns the SQL into MapReduce code then runs on our cluster. We can analyse the large data using **Pig**. **Impala**(optimized for low latency queries..so faster than Hive) is developed as a way to query the data with SQL, but which directly accesses the data in HDFS without the help of MapReduce. **Sqoop** takes the data from the traditional relational database and puts it in HDFS, so it can be processed along with other data on the cluster. **Flume** injests the data as it's generated by external systems and puts it in HDFS. **HBase** is the realtime database built on top of HDFS. **Hue** is a graphical front end to the cluster. **Oozie** is a workflow management tool. **Mahout** is a machinelearning library. A company Cloudera offers **CDH**, a complete Ecosystem.             

### What's HDFS?
See what's going on behind the scenes when you store the data
<img src="https://user-images.githubusercontent.com/31917400/44589197-ccaff880-a7af-11e8-8989-9bb3e89718db.jpg" />

 - When a file is loaded into HDFS, it's splittd into chunks called **blocks**. Each block is still big(the default is **64mb**) and given an unique name such as **blk_1, blk_2..**. For example, assuming my file is 150 mb, the first and second block is 64mb, 64mb, then the third block is 22mb. As the file is uploaded, each block will get stored on one **node** in the cluster. 
   - There is a software called **Daemon** running on each of the machine in the cluster and we call them the **DataNode**(slave). 
   - We need to know which blocks make up the original file. That is handled by separate machine, running Daemon called the **NameNode**(master).
   - The information is stored on the **NameNode** is called **metadata**.
   
> [Daemon]
 - Before running `MapReduce`, we submit the job to what's called the **Job-tracker**(Namenode) which splits the work into **Mappers & Reducers**(running on Datanodes).
 - `Running the actual Map` and `Reducing task` is handled by a **task-tracker**(Daemon), a software running on each of these nodes. Since the **task-tracker** runs on the same machine as the Datanode, the Hadoop framework will be able to have the **Map_tasks** work directly on the pieces of data sorted on that machine which saves lots of network traffic. 
 - By default, Hadoop uses an **HDFS block** as the input-split for each Mapper. It'll try to make sure Mappers work on the data on the same machine. Once HDFS splits the data for each Mapper or Datanode, lets Mappers work in parallel to respond to their master coz this broken pieces are all from one data.   
 
> [failure]
 - In case of Data-Node-failure, Hadoop replicates each block **3 times**. so if a single node fails, it's ok coz we have 2 other copies of the block on other nodes. 
 - The **Namenode** is smart enough to rearrange to have those blocks re-replicated on the cluster. but what if the Name-Node fails, burst into flame? 
   - **data, entire cluster become inaccessible**.
   - **we lost the metadata forever** (we still have blocks but we can't see which block belongs to which file).
   - That's why we configure our **Namenode** not only on its own hard drive, but also somewhere on a network file system(NFS) which is a method of mounting a remote disk. As a better alternative, people configure two **Namenodes** so that it is not a single point of failure in most production clusters.
     - Active Name-Node
     - Standby Name-Node 
     
That way, our cluster can keep running.

> [MapReduce]: data processing.
 - Every Datanode(server) has a Mapper(finding) and Reducer(processing). Let's say, we have a big data with massive rowssssss. And we have 5 servers.
 - Suppose we have a task: `Counting how many times the word 'korea' appears!` 
 - The Namenode(my laptop with **job-tracker**) first starts [Mappers], i.e, the Namenode first requests datanodes(servers) run their Mappers(for example "find korea and record"!) then Mappers in each Datanode write a new file(`mapper_output.txt`) whenever they find things. Since we have 5 servers(Datanodes), we would get 5 `mapper_output.txt` files...and Done!    
 - The Namenode(my laptop with **job-tracker**) then starts [Reducers], i.e, the Namenode requests datanodes(servers) run their Reducers(for example, "count the record"!) then Reducers in each Datanode read the new file(`mapper_output.txt`) and do some calculation and output the final answer. 
 - Historically, we use an associative array called **Hash_table** that consists of **key** and **value**. The problem is that if we run some terabyte of data...it'll take too long time and run out of memory holding the hash_table. 
 - So...our [Mappers] take the ledgers, break it into chunks, give each chunk to one of our [Mappers]. **All Mappers at the same time are 1.working each of small fraction of the data, writing `index cards`, 2.piling up hash_tables so that cards for the `same key` go in the same pile.** 
 - Once the Mappers have finished, our [Reducers] collect these sets of card(for example .txt file) and the Master(Namenode) tells our Reducers which key they are responsible for. The [Reducers] go to the Mappers and retrieve the pile of cards for their keys, then collect all the small piles and add up to larger piles. This is followed by going through(alphabetical order) the piles one at a time and process in some way all values from all the card on the pile.
<img src="https://user-images.githubusercontent.com/31917400/44621428-48886e80-a89e-11e8-9224-4247dde159c6.jpg" />

So...in summary,
 - 1> [Mappers] are just little programs(coded algorithm) that each deals with a small data, working in parallel. We call their outputs as **intermediate_records** and writing them on the **index cards**(Hadoop deals with all data in the form of **key** and **value**). 
 - 2> Then **Shuffle and Sort** takes place.
   - Shuffle: the movement of the intermediate records from the Mappers to the Reducers
   - Sort: Reducers organize these sets of records into sorted order (Hadoop takes care of the Shuffle and Sort phase. You do not have to sort the keys in your reducer code, you get them in already sorted order). 
 - 3> Finally, each [Reducer] works on one set of records(one pile of cards) at a time, gets the key and the list of all the values, then sort(produce) our final result.

### Example(writing mapreduce engine) 
 - Check if we have our input data in HDFS: `hadoop fs -ls [directory]`
 - Check if we have our output data in HDFS: `hadoop fs -ls [directory]` 
 - Check our Mapper, Reducer code file: `ls`
 - To submit our job: `hadoop jar [path_to_jar] -mapper [mapper_file] -reducer [reducer_file]` then specify the input directory in HDFS `-file [mapper_file] -file [reducer_file]` then specify the output directory to which the reducers will write their output data. `-input [directory] -output [directory]`
 - take a look at the contents of our output(generated from the reducer): `hadoop fs -cat [output] | less`
 - To retrieve the data from HDFS and put it onto our local machine: `hadoop fs -get [output] [input_file]`
 
> Example
 - __Mapper__(mapper.py)
   - Each mapper processes a portion of the input data, and each one will be given a line(record) at a time. The mapper take the line and extract the information it needs. It often uses RegExpression.
   - mapper code in python: when we can understand the line via tab(\t)- we called 'tab_delimited'(we want...the total sales per store?)
   - But we often encounter the weirdness of the massive dataset such as mal-formed lines..then mapper dies and we will be on halfway through terabyte jobs so we need to make sure that no matter what kind of data we get, the mapper can continue working: We call it defensive programming...for example....
```
def mapper():
    import sys
    
    for i in sys.stdin:
        data = i.strip().split("\t")
        if len(data)==6:
            date, time, store, item, cost, payment = data
            print("{0}\t{1}".format(store, cost))
```
 - What happens b/w mappers and reducers?: Shuffle and Sort
   - Ensuring the values for any particular key are collected together, then sending the keys and their list of values to the reducer.  
 - __reducer__(reducer.py)
   - let's say we have a single reducer which is the default in Hadoop, so it will get all the keys. If we had specified more reducer, each would receive some of the keys along with the values for those particular keys.
   - We use 'Hadoop Streaming' to write a code in python. Well, keys are already sorted, then what variables do we need to keep track of? 
```
def reducer():
    import sys
    
    sales_total = 0
    old_key = None
    
    for i in sys.stdin: ##### perhaps..such as..["Miami 12.34","Miami 99.07","Miami 55.07","NYC 88.97","NYC 33.56"]
        data = i.strip().split("\t")
        if len(data) != 2:
            continue ##### Something has gone wrong. Skip this line.
            
        thisKey, thisSale = data
        
        if i in old_key != thisKey:
            print "{0}\t{1}".format(old_Key, sales_total)
            sales_total = 0
        old_key = thisKey
        sales_total += float(thisSale)
    
    if old_key != None:
        print "{0}\t{1}".format(old_key, sales_total)
```
In Hadoop, one of the nice thing about using "Hadoop Streaming" is that it's easy to test our code outside of Hadoop.
 - Our mapper takes input from **standard input**.  

### MapReduce Design Patterns
<img src="https://user-images.githubusercontent.com/31917400/44629611-bd64b280-a949-11e8-9e90-2222075881e3.jpg" />




-------------------------------------------------------------------------------------------------
# Chapter 01. Data Modelling
> When to use RDBS?
 - Need Flexibility for writing in SQL queries: With SQL being the most common database query language.
 - Need Modeling the data not modeling queries
 - Need Ability to do JOINS
 - Need Ability to do aggregations and analytics
 - Need Secondary Indexes available : You have the advantage of being able to add another index to help with quick searching.
 - Smaller data volumes: If you have a smaller data volume (and not big data) you can use a relational database for its simplicity.
 - Need ACID Transactions: Allows you to meet a set of properties of database transactions intended to guarantee validity even in the event of errors, power failures, and thus maintain data integrity.
 - Need Easier to change to business requirements
 - __DON'T USE WHEN YOU__
   - Have large amounts of data: Relational Databases are not distributed databases and because of this they can only scale vertically by adding more storage in the machine itself. You are limited by how much you can scale and how much data you can store on one machine. You cannot add more machines like you can in NoSQL databases.
   - Need fast reads and write.
   - Need to be able to store different data type formats: Relational databases are not designed to handle unstructured data.
   - Need high throughput -- fast reads: While ACID transactions bring benefits, they also slow down the process of reading and writing data. If you need very fast reads and writes, using a relational database may not suit your needs.
   - Need a flexible schema: Flexible schema can allow for columns to be added that do not have to be used by every row, saving disk space.
   - Need high availability: The fact that relational databases are not distributed (and even when they are, they have a coordinator/worker architecture), they have a single point of failure. When that database goes down, a fail-over to a backup system occurs and takes time. so the system is not always up and there is downtime. 
   - Need horizontal scalability: Horizontal scalability is the ability to add more machines or nodes to a system to increase performance and space for data.

> When to use NoSQL?
 - Need to be able to store different data type formats: NoSQL was also created to handle different data configurations: structured, semi-structured, and unstructured data. JSON, XML documents can all be handled easily with NoSQL.
 - Large amounts of data: Relational Databases are not distributed databases and because of this they can only scale vertically by adding more storage in the machine itself. NoSQL databases were created to be able to be horizontally scalable. The more servers/systems you add to the database the more data that can be hosted with high availability and low latency (fast reads and writes).
 - Need horizontal scalability: Horizontal scalability is the ability to add more machines or nodes to a system to increase performance and space for data
 - Need high throughput: While ACID transactions bring benefits they also slow down the process of reading and writing data. If you need very fast reads and writes using a relational database may not suit your needs.
 - Need a flexible schema: Flexible schema can allow for columns to be added that do not have to be used by every row, saving disk space.
 - Need high availability: Relational databases have a single point of failure. When that database goes down, a failover to a backup system must happen and takes time.
 - __DON'T USE WHEN YOU__
   - When you have a small dataset: NoSQL databases were made for big datasets not small datasets and while it works it wasn’t created for that.
   - When you need ACID Transactions: If you need a consistent database with ACID transactions, then NoSQL databases will not be able to serve this need. NoSQL database are eventually consistent and do not provide ACID transactions.
   - When you need the ability to do JOINS across tables: NoSQL does not allow the ability to do JOINS. This is not allowed as this will result in a full table scans.
   - When you want to be able to do aggregations and analytics
   - When you have changing business requirements : Ad-hoc queries are possible but difficult as the data model was done to fix particular queries
   - When your queries are not available and you need the flexibility : You need your queries in advance. If those are not available or you will need to be able to have flexibility on how you query your data you might need to stick with a relational database.
   
## > PostgreSQL Basic: 1) `autocommit = True`
 - 1. **Connect** to the local instance of PostgreSQL (127.0.0.1)
 - 2. **Get a cursor** that will be used to execute queries
 - 3. Create a database to work in
 - 4. One action = one transaction...means we should run "commit" each transaction or getting a strange error.
   - having to call `conn.commit()` after each command. The ability to rollback and commit transactions are a feature of Relational Databases. 
 - 5. If you don't want, then do `autocommit` !!
```
## import postgreSQL adapter for the Python
import psycopg2

conn = psycopg2.connect("host=127.0.0.1 dbname=studentdb user=student password=student")
cur = conn.cursor()
# for example
cur.execute("select * from old_table")
conn.commit()
# but...
conn.set_session(autocommit=True)
cur.execute("CREATE TABLE new_table (col1 int, col2 int, col3 int);")
cur.execute("select count(*) from new_table")
print(cur.fetchall())
```
 - 5. we can create new database as well.
```
try: 
    cur.execute("create database kimdb")
except psycopg2.Error as e:
    print(e)
```

## > PostgreSQL Basic: 2) `conn.close()` 
 - 0. Close our connection to the default database, reconnect to the kimdb database, and get a new cursor.
```
try: 
    conn.close()
except psycopg2.Error as e:
    print(e)
  
try: 
    conn = psycopg2.connect("host=127.0.0.1 dbname=kimdb user=student password=student")
except psycopg2.Error as e: 
    print("Error: Could not make connection to the Postgres database")
    print(e)
    
cur = conn.cursor()
conn.set_session(autocommit=True)
```
 - 1. create a table...`IF NOT EXISTS`, `DROP table`,..
   - Table Name: music_library
   - column 1: Album Name
   - column 2: Artist Name
   - column 3: Year
 - 2. Insert rows of data
   - If you run the insert statement code more than once, you will see **duplicates** of your data.
 - 3. Validate the information
 - 4. Drop the table to avoid duplicates and clean up
```
#1.
cur.execute("CREATE TABLE IF NOT EXISTS music_library (album_name varchar, artist_name varchar, year int);")
#2.
cur.execute("INSERT INTO music_library (album_name, artist_name, year) VALUES (%s, %s, %s)", ("Let It Be", "The Beatles", 1970))
cur.execute("INSERT INTO music_library (album_name, artist_name, year) VALUES (%s, %s, %s)", ("Rubber Soul", "The Beatles", 1965))
#3.
cur.execute("SELECT * FROM music_library;")
row = cur.fetchone()

while row:
    print(row)
    row = cur.fetchone()
#4.
cur.execute("DROP table music_library")
cur.close()
conn.close()
```
## > Cassandra Basic: `session = cluster.connect()`
 - 1. Create a connection to the database
 - 2. Create a keyspace to the work in and connect to the keyspace
 - 3. Create a table(translate this information below into a Create Table Statement)
   - Table Name: music_library
   - column 1: Album Name
   - column 2: Artist Name
   - column 3: Year 
   - PRIMARY KEY(year, artist name)
 - 4. Insert rows of data
 - 5. validate the information
 - 6. Drop the table to avoid duplicates and clean up
```
from cassandra.cluster import Cluster
#1.
clu = Cluster(['127.0.0.1'])
# the connection_object
session = clu.connect() 
# no need to have cursor!!! or open session..or 
session.execute("""select * from music_libary""")
#2.
session.execute("""CREATE KEYSPACE IF NOT EXISTS test_keyspace WITH REPLICATION = { 'class' : 'SimpleStrategy', 'replication_factor' : 1 }""")
session.set_keyspace('test_keyspace')


#3.
query = "CREATE TABLE IF NOT EXISTS music_library "
query = query + "(year int, artist_name text, album_name text, PRIMARY KEY (year, artist_name))"
try:
    session.execute(query)
except Exception as e:
    print(e)
#4.
query = "INSERT INTO music_library (year, artist_name, album_name)"
query = query + " VALUES (%s, %s, %s)"
session.execute(query, (1970, "The Beatles", "Let it Be"))
session.execute(query, (1965, "The Beatles", "Rubber Soul"))
#5.
query = 'SELECT * FROM music_library'
rows = session.execute(query)
for row in rows:
    print (row.year, row.album_name, row.artist_name)
#6.
query = "DROP table music_library"
rows = session.execute(query)
session.shutdown()
cluster.shutdown()
```
[For SQL]---------------------------------------------------------------------------------------------------------------------------
## ACID theorem:
 - > 1.Atomicity: 
   - All components of a transaction are treated as a single action. All are completed or none are; if one part of a transaction fails, the database’s state is unchanged.
 - > 2.Consistency: 
   - Transactions must follow the defined rules and restrictions of the database, e.g., constraints, cascades, and triggers. Thus, any data written to the database must be valid and any transaction that completes will change the state of the database. No transaction can create an invalid data state. Note that this is different from “consistency” as it’s defined in the CAP theorem.
 - > 3.Isolation: 
   - Fundamental to achieving concurrency control, isolation ensures that the concurrent execution of transactions results in a system state that would be obtained if transactions were executed serially, i.e., one after the other. With isolation, an incomplete transaction cannot affect another incomplete transaction.
 - > 4.Durablity: 
   - Once a transaction is committed, it will persist and will not be undone to accommodate conflicts with other operations. Many argue that this implies the transaction is on disk as well; most formal definitions aren’t specific.

## > Structuring database with SQL(i)--(Normalization:`break`) / Denormalization:`JOIN more or less`)
> Normalization(you will feel natural): `Faster Writing!`
 - To Free the database from unwanted insertrions, updates, deletion, etc.
   - Reduce **data redundancy**
     - (kill copies) 
 - To Reduce the need for refactoring the database as new data are introduced???
 - To Make the database neutral to the query statistics(NOT to focus on a particular query).
   - Increase **data integrity**
     - (increase the likelihood that data is correct in all locations) 
 - Normal form(1NF, 2NF, 3NF)
   - 1NF: Atomic values
   - 2NF: All columns in the table must rely on the **Primary Key**
     - primary: unique
     - foreign: Not unique, but it can be primary for other tables.
   - 3NF: No transitive dependencies
     - When getting from A-> C, you want to avoid going through B.
     - we use 3NF because when updating data, we want to be able to do in just 1 place.

> Denormalization(you will feel unnatural): `Faster Reading!`
 - To Increase performance in case of heavy **READING** workload..
   - Duplicate the copies of data for some reason such as JOINS? 
   - JOINS on the database allow for outstanding flexibility but are extremely slow. If you are dealing with heavy reads on your database, you may want to think about denormalizing your tables. The denormalization comes after normalization.
 - We will have **full information** table specific to a particular topic.   

## > Structuring database with SQL(ii)--(star & snowflake)
 - Star schema is the simplest style of **Data Mart**. It can consist of multiple **Fact_tables**(at the center) referencing any number of dimension tables (but it decrease the query flexibility).
   - To get simpler queries 
   - To denormalize
   - To get faster aggregation
 - Snowflake schema is more normalized version of Star schema(but only in 1NF/2NF)

[For NoSQL]-------------------------------------------------------------------------------------------------------------------------
## CAP Theorem:
 - > 1.Consistency: 
   - Every read from the database gets the latest (and correct) piece of data or an error
 - > 2.Availability: 
   - Every request is received and a response is given (without a guarantee that the data is the latest update)
 - > 3.Partition Tolerance: 
   - The system continues to work regardless of losing network connectivity between nodes 

## > Structuring database with Cassandra(i)--(Denormalization:`There are no JOIN, GROUP BY ??`)
 - Denormalization is not just okay..**it's a must**. Denormalization must be done for fast reads
 - Apache Cassandra has been optimized for **fast writes**
 - ALWAYS think Queries first. It does not allow for JOINs between tables. ASK first: `What queries will be perfomed on that data?`
 - **One table per query** is a great strategy

> In Apache Cassandra, if your business need calls for quickly changing requirements, you need to create a new table to process the data. If your business needs calls for ad-hoc queries, these are not a strength of Apache Cassandra. However keep in mind that it is easy to create a new table that will fit your new query

## > PRIMARY KEY(unique but simple/composite) in Cassandra
 - The **PRIMARY KEY** is made up of either:
   - just the `PARTITION KEY` or 
   - may also include additional `CLUSTERING COLUMNS`.
 - A Simple **PRIMARY KEY** is just one column that is also the `PARTITION KEY`. 
 - A Composite **PRIMARY KEY** is made up of more than one column and will assist in creating a unique value and in your retrieval queries.
   - The `PARTITION KEY` will determine the distribution of data across the system.
   - The `clustering column` will sort the data in sorted ascending order(or alphabetical) in the table.
   - More than one `clustering column` can be added (or none!).
   - From there, the `clustering columns` will sort in order of **how they were added to the primary key**.
   <img src="https://user-images.githubusercontent.com/31917400/60138535-a3cb0c00-97a2-11e9-959a-ce4b0828b213.jpg" />

     - You can use as many `clustering columns` as you would like. You cannot use the `clustering columns` out of order in the **SELECT statement**. You may choose to omit using a `clustering column` in your **SELECT statement**. That's OK. Just remember to use them in order when you are using the **SELECT statement**.

## > Apache Cassandra does not allow for duplicated data in the rows.


-------------------------------------------------------------------------------------------------
# Chapter 02. Data WareHousing
> Perspective 01- **Business** (if you are in charge of a retailer’s data infrastructure?)
 - See some business activities:
   - Customers should be able to find goods & make orders
   - Inventory Staff should be able to stock, retrieve, and re-order goods
   - Delivery Staff should be able to pick up & deliver goods
   - HR should be able to assess the performance of sales staff
   - Marketing should be able to see the effect of different sales channels
   - Management should be able to monitor sales growth
 - Can we build a single database to support these activities? Are all of the above questions of the same nature? 
 ## NOPE.
<img src="https://user-images.githubusercontent.com/31917400/58917762-97adda80-871f-11e9-8114-6a340bb1e46d.jpg" />

> Perspective 02 - **Technical** 
- What is DWH? 
 - DWH is a `copy` of transaction data specifically **structured for** `query and analysis`. 
 - DWH is subject-oriented(categorized by topic), integrated(coming from many sources), non-volatile(non-transient), time-variant(changing questions by time) collection of data in support of management's decisions. When the data is so large and diverse, databases cannot handle them because its too expensive, hard to query...we consider DWH?
 - DWH is a system retrieving and consolidating data periodically from the source systems into a dimensional, normalized data store. It keeps years of history. It is typically updated in batches, not every time a transaction happens in the source system. 
<img src="https://user-images.githubusercontent.com/31917400/58919046-97fca480-8724-11e9-91db-6c44fde98fa1.jpg" />

## Now we store our tables `after ETL` into a **dimensional model**(for analytics). 
 - What's the dimensional, normalized store? 
   - **Dimensional modeling** has two goals 
     - 1. easy to understand?
     - 2. faster analytical query? 
     <img src="https://user-images.githubusercontent.com/31917400/59024835-b230a280-884a-11e9-8493-b2595bfe0f68.jpg" />

   - Love star? then define first which is `dimension` / `fact`. And create **Dimension_table** and **Fact_table**.
     - Dimension_table(Quality): 
       - Context(Attribute): who(customer name?), when(data or time), where(store name?), what(product name?),..
     - Fact_table(Quantity): 
       - Record in quantifiable metrics: quantity, duration, rate,..(explaining events numerically)
       - Ask "Is it additive?"... does the additive have meanings? If not, it's not a fact. 
     <img src="https://user-images.githubusercontent.com/31917400/59028364-61717780-8853-11e9-9720-587448b1ce76.jpg" />

 - `Naive ETL`=> move From **3NF** to **Star**
   - Extract: Query the 3NF database
   - Transform:
     - JOIN tables
     - Change data types
     - Add new features
   - Load: Structure and load the data into the dimensional data model(Fact / Dimension)

## Data Warehouse Architecture Examples
- User: Front_Room
- Data engineer: Back_Room(ETL process)

 - > 1.Kimball's Bus: `User` cannot decide the schema organization
 - > 2.Data Marts: `User` can decide the schema organization
 - > 3.Inmon's Corporate Information Factory (CIF): `User` can decide the schema organization
 - > 4.Hybrid of [Bus + CIF]: `User` can decide the schema organization

DWH architecture varies depends on the answer of this question: `To what extent is data engineer(you) gonna let USERS decide how the data schemas are organized?`: The answer will ultimately decides different **ETL method** in the Back_Room - such as the way data stored.

### 1. Kimball's Bus: 
<img src="https://user-images.githubusercontent.com/31917400/59796328-218ba500-92d5-11e9-9c97-37727eacae75.jpg" />
  
 - It results in a **common dimension data model** shared by different departmentsss!! (so 'sales analytics' and the 'delivery analytics' will both use the same data dimension). 
 - Data is kept at the **atomic level**, not at the aggregated level.
 - The **bus matrix** is given to Users?????  
 
 - `Kimball's ETL` => Users cannot access the Back_Room work.
   - Extract:
     - Get the data from its source
     - `Delete old state`
   - Transform:
     - `Integrate many sources together.`
     - `Produce diagnostic **metadata**.`
   - Load: Structure and load the data into the dimensional data model(Fact / Dimension). 

  
### 2. Data Marts:
<img src="https://user-images.githubusercontent.com/31917400/59930327-a0e6b900-943a-11e9-9bda-9439ab86bed3.jpg" />

 - Each department(User) has to deal with ETL directly without Data engineer's help. **NOT RECOMMENDED!!!**
 - Anti-conformed dimension!: Under the hood, they would repeat each other's work, model the dimension differently...
   - different Fact_table for the same event due to no-conformed dimension.
   - Independent ETL, Dimensional model..so it will produce smaller separated dimension models.
   - It emerged from the idea of `departmental autonomy`, but their uncoordinated effort can lead to inconsistent views. 
   
 - `Data Mart's ETL` => varies by each department!

### 3. Inmon's CIF
<img src="https://user-images.githubusercontent.com/31917400/59930382-b52ab600-943a-11e9-88a6-581fd8d4f07e.jpg" />

 - `Enterprise DWH` refers **Normalized part** in the CIF architecture. It can be accessed by END-Users if necessary. 
 - It uses **Data Marts**! but they are already coordinated by `Enterprise DWH`. so..`departmental autonomy` works here!
 - Unlike Kimball's model, data can be kept at the aggregated level. 
 
 - `Inmon's ETL` => There are 2 ETL processes required here. 
   - 1) Source transaction -> 3NF database(Enterprise DWH)
   - 2) 3NF database(Enterprise DWH) -> Departmental Data Marts  

### 4. Bus + CIF
<img src="https://user-images.githubusercontent.com/31917400/59800371-67993680-92de-11e9-8e4f-388021683bc2.jpg" />

 - W/O Data Marts!


## At the end of the day, we want OLAP to `query`.  
 - Online **Analytical** Process (OLAP) Cube
<img src="https://user-images.githubusercontent.com/31917400/59842654-e0d07200-934e-11e9-83e9-cc7195e2a646.jpg" />

0> How to serve OLAP cube?
 - method_A (**M..OLAP**): pre-aggregate OLAP cubes and save them on **special purpose non-relational database** ..Buy OLAP server?  
 - method_B (**R..OLAP**): compute OLAP on the fly from the existing Relational Database where the dimensional models reside. 

1> OLAP cube is an **aggregation** of a "fact metric" on a number of dimensions(by taking a combination of dimensions such as movie, country, month). It makes things easy to communicate to business(end) users. Once you build the cube, how to address them?  
 - General Operations: `Roll-Up`, `Drill-Down`, `Slice&Dice`
 - __Roll-Up__: Aggregates or combines values and reduces number of rows or columns.
 - __Drill-Down__: Decomposes values and increases number of rows or columns.
 - __Slice__: Reducing N dimensions to N-1 dimensions by restricting one dimension to a single value (same cube with thinner depth)
 - __Dice__: Same dimensions but computing a sub-cube by restricting, some of the values of the dimensions (smaller cube with same depth)

2> OLAP cube query optimization
 - Typically, the operations in real world setting are quite ad-hoc. 
 - `Group by CUBE(dim01, dim02,..)` makes one dimension **pass through the Fact_table** and aggregates all possible combinations of groupings...=> No need to process the whole Fact_tables again and again. 
 - forthcoming aggregations:
   - total_dim(k)
   - dim(k)by `dim01` -> dim(k)by `dim02` -> dim(k)by `dim03`....
   - dim(k)by `dim01&dim02` -> dim(k)by `dim02&dim03`, ....
   - dim(k)by `dim01&dim02&dim03&...`






 








































