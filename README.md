# Study-Business-Analytics-II-Practice
HDFS, MapReduce, Apache Spark PostgreSQL

---------------------------------------------------------------------------------------------------------------------------
A relation means structure. Relational Database(SQL) is suitable for realtime crud(create/read/update/delete) operation while a big data stack(Hadoop) stores both structured/unstructured data without need of relations, consistency of format, etc. 

### Intro to MapReduce and Hadoop
Hadoop splits the data up and stores it across the collection of machines(a cluster) then processes the data in place **where it's actually stored(within the cluster)** rather than retrieving the data from a central server. Of course we can add more machines to the cluster as the amount of the data we storing grows(machines in the cluster is typically `mid-range servers`).
 - MapReduce is for processing
 - HDFS is for storing

> Hadoop_EcoSystem:
<img src="https://user-images.githubusercontent.com/31917400/44554202-f7566e80-a727-11e8-862c-a3a2d034549c.jpg" />

In addition to **Core Hadoop**(HDFS+MapReduce), other software has grown up around it to make Hadoop easier to use. For example, Writing MapReduce is not simple and this is where Hive or Pig come into play. In **Hive**, we can just write SQL-like statement, and Hive interpreter turns the SQL into MapReduce code then runs on our cluster. We can analyse the large data using **Pig**. **Impala**(optimized for low latency queries..so faster than Hive) is developed as a way to query the data with SQL, but which directly accesses the data in HDFS without the help of MapReduce. **Sqoop** takes the data from the traditional relational database and puts it in HDFS, so it can be processed along with other data on the cluster. **Flume** injests the data as it's generated by external systems and puts it in HDFS. **HBase** is the realtime database built on top of HDFS. **Hue** is a graphical front end to the cluster. **Oozie** is a workflow management tool. **Mahout** is a machinelearning library. A company Cloudera offers **CDH**, a complete Ecosystem.             

> [HDFS]: See what's going on behind the scenes when you store the data
<img src="https://user-images.githubusercontent.com/31917400/44589197-ccaff880-a7af-11e8-8989-9bb3e89718db.jpg" />

 - When a file is loaded into HDFS, it's splittd into chunks called **blocks**. Each block is still big(the default is **64mb**) and given an unique name such as **blk_1, blk_2..**. For example, assuming my file is 150 mb, the first and second block is 64mb, 64mb, then the third block is 22mb. As the file is uploaded, each block will get stored on one **node** in the cluster. 
   - There is a software called **Damon** running on each of the machine in the cluster and we call them the **Data-Node**. 
   - We need to know which blocks make up the original file. That is handled by separate machine, running **Damon** called the **Name-Node**.
   - The information is stored on the **Name-Node** is called **metadata**.

> [failure]__
 - In case of Data-Node-failure, Hadoop replicates each block 3 times. so if a single node fails, it's ok coz we have 2 other copies of the block on other nodes. 
 - The Name-Node is smart enough to rearrange to have those blocks re-replicated on the cluster. but what if the Name-Node fails, burst into flame? 
   - **data, entire cluster become inaccessible**
   - **we lost the metadata forever** (we still have blocks but we can't see which block belongs to which file)
   - That's why we configure our **Name-Node** not only on its own hard drive, but also somewhere on a network file system(NFS) which is a method of mounting a remote disk. As a better alternative, people configure two **Name-Nodes** so that it is not a single point of failure in most production clusters
     - Active Name-Node
     - Standby Name-Node 
     
That way, our cluster can keep running.

> [MapReduce]: See how the data is processed.
 - Historically, we use an associative array called **Hash_table** that consists of **key** and **value**. The problem is that if we run some terabyte of data...it'll take too long time and run out of memory holding the hash_table. 
 - So...our Mappers take the ledgers, break it into chunks, give each chunk to one of our **Mappers.** All Mappers are working at the same time each of small fraction of the data, writing **index cards**, piling up hash_tables so that cards for the same key go in the same pile. Once the Mappers have finished, our Reducers collect these sets of card and we tell our Reducers which key they are responsible for. The Reducers go to the Mappers and retrieve the pile of cards for their keys, then collect all the small piles and add up to larger piles. This is followed by going through(alphabetical order) the piles one at a time and process in some way all values from all the card on the pile.
<img src="https://user-images.githubusercontent.com/31917400/44621428-48886e80-a89e-11e8-9224-4247dde159c6.jpg" />

So...in summary,
 - Mappers are just little programs that each deals with a small data, working in parallel. We call their outputs as **intermediate_records** and writing them on the **index cards**(Hadoop deals with all data in the form of **key** and **value**). 
 - Then **Shuffle and Sort** takes place.
   - Shuffle: the movement of the intermediate records from the Mappers to the Reducers
   - Sort: Reducers organize these sets of records into sorted order (Hadoop takes care of the Shuffle and Sort phase. You do not have to sort the keys in your reducer code, you get them in already sorted order). 
 - Finally, each reducer works on one set of records(one pile of cards) at a time, gets the key and the list of all the values
 - Then sort our final result .. how to get our final result in a sorted order?
   - need only one reducer
   - need an extra step to merge the result

> [Daemon]
 - When running MapReduce job, we submit the job to what's called the **Job-tracker** which splits the work into **Mappers & Reducers**. Those Mappers and Reducers will run on the other cluster nodes. 
 - Running the actual **Map & Reduce** task is handled by a daemon called the **task-tracker**, a software running on each of these nodes. Since the task-tracker(daemon) runs on the same machine as the Data-Node, the Hadoop framework will be able to have the **Map_tasks** work directly on the pieces of data sorted on that machine which saves lots of network traffic. 
 - By default, Hadoop uses an **HDFS block** as the input split for each Mapper. It'll try to make sure a **Mapper** works on the data on the same machine. But what about Reducer??

> [write Mapper]
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






