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



-------------------------------------------------------------------------------------------------
# Chapter 02. Data WareHousing
> Perspective 01- **Business** (if you are in charge of a retailer’s data infrastructure?
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

1> It is an **aggregation** of a "fact metric" on a number of dimensions(by taking a combination of dimensions such as movie, country, month). It makes things easy to communicate to business(end) users. Once you build the cube, how to address them?  
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






 








































