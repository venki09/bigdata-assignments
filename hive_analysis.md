
## Hive Analysis
The purpose of this document is to list out all the steps done as part of hive analysis

## Hive setup
For the purpose of this assignment, I used the cloudera quickstart docker image. Using a docker image has the following advantages
* Not dependent on the host platform. It can run anywhere
* The user doesn't have to worry about any additional setup to configure nodes and things like that.
* Setup and teardown is very easy and can be done with one command

## Setup steps
* Install cloudera docker image which has all the cloudera hadoop distribution for us to work with.
```
docker pull cloudera/quickstart:latest
```
* Run the docker container which will bring up the cluster and start all the hadoop services.
```
docker run --hostname=quickstart.cloudera --privileged=true -t -i -p 8888 -p 7180 4239cd2958c6 /usr/bin/docker-quickstart
```
* As soon as the docker container starts, it will open up the terminal which we can use to access the filesystem in that container.

* Start hive metastore server. This needs to be done so that hive can contact the metastore to perform all the operations.

* Run it as background process so that we can do other tasks with the hive metastore thrift server started.
```
hive --service metastore &
```
* Now we can start hive console using hive command `hive` on the terminal. Another way to access hive is to use a cloudera provided service called hue which is a UI component which exposes the following hadoop services for us
   * Hive editor
   * HDFS file browser
   * Job tracker and so on..

## Load datasets into Hive
Fo the purpose of this assignment, we will be analysing weather data provided here in https://catalog.data.gov/dataset?res_format=CSV&tags=weather. We will do some simple analysis like days where there were no rain, Coldest and hottest days etc. Following are the steps taken to load the datasets into HDFS and query them using hive.

* Now we need to load data into HDFS so that we can use it to create hive tables
* Install wget in container so that we can get dataset from https://gist.githubusercontent.com/venki09/15e48ae1b96b3e38fc3ea20080aaf9a1/raw/e55e58a92d03e2b29f25175cd0416a42cbc1ebe2/weather.json which is taken from https://catalog.data.gov/dataset?res_format=CSV&tags=weather. 
* Download dataset using 
```
https://gist.githubusercontent.com/venki09/15e48ae1b96b3e38fc3ea20080aaf9a1/raw/e55e58a92d03e2b29f25175cd0416a42cbc1ebe2/weather.json which is taken from https://catalog.data.gov/dataset?res_format=CSV&tags=weather
```
* Upload the json file to HDFS
```
hdfs dfs -mkdir /hive_datasets # creating a directory
hdfs dfs -copyFromLocal weather.json /hive_datasets # copy from local
```
<Screenshot goes here!>
* To create hive table with json SerDe(Serializer Deserializer), we need to download the hcatalog jar which supports that and then add it to the hive path so that our query can recognize it.
```
wget wget http://central.maven.org/maven2/org/apache/hive/hcatalog/hcatalog-core/0.12.0/hcatalog-core-0.12.0.jar
hdfs dfs -mkdir /jars #Create jars directory to throw in all dependency jars needed
hdfs dfs -copyFromLocal hcatalog-core-0.12.0.jar /jars
```
* Open up the hive console and add the hcatalog jar
```
$ hive
hive> ADD JAR hdfs:///jars/hcatalog-core-0.12.0.jar;
```
From here on, I have started using the cloudera provided hive query editor called hue which is easier to visualize data.
* Create table command
```
CREATE EXTERNAL TABLE IF NOT EXISTS weather(
        docid INT,
        weatherdetails array<struct< 
        id: INT,
        fogground: STRING,
        snowfall: DOUBLE,
        dust: STRING,
        snowdepth: DOUBLE,
        mist: STRING,
        drizzle: STRING,
        hail: STRING,
        fastest2minwindspeed: DOUBLE,
        thunder: STRING,
        glaze: STRING,
        snow: STRING,
        ice: STRING,
        fog: STRING,
        temperaturemin: DOUBLE,
        fastest5secwindspeed: DOUBLE,
        temperaturemax: DOUBLE,
        freezingfog: STRING,
        blowingsnow: STRING,
        freezingrain: STRING,
        rain: STRING,
        highwind: STRING,
        date: STRING,
        precipitation: DOUBLE,
        fogheavy: STRING,
        smokehaze: STRING,
        avgwindspeed: DOUBLE,
        fastest2minwinddir: DOUBLE,
        fastest5secwinddir: DOUBLE>>)
    ROW FORMAT 
    SERDE 'org.apache.hive.hcatalog.data.JsonSerDe'
    STORED AS TEXTFILE
    location 'hdfs:///hive_datasets';
```
* Describe table to make sure we have all the columns
```
hive> desc 'weather';
```
* Select some rows from table to make sure we have all the data loaded properly
```
hive> select * from weather;
```
<Screenshot goes here>
  
## Analysis
### Days where there was no rain
```
select weatherdetail.id from weather lateral view explode(weatherdetails) weathertable as weatherdetail
where weatherdetail.rain=='No';
```
### Days where there was no rain but was foggy
```
select weatherdetail.id from weather lateral view explode(weatherdetails) weathertable as weatherdetail
where weatherdetail.rain=='No' AND weatherdetail.fog=='Yes';
```
### Top 5 coldest days in descending order
```
select weatherdetail.id,weatherdetail.temperaturemin from weather 
lateral view explode(weatherdetails) weathertable as weatherdetail 
order by temperaturemin ASC
limit 5
```
### Top 5 hottest days in descending order
```
select weatherdetail.id,weatherdetail.temperaturemax from weather 
lateral view explode(weatherdetails) weathertable as weatherdetail 
order by temperaturemax ASC
limit 5
```

## Some problems that I ran into
There were a few issues that I ran into while performing this assignment. Some of them are listed below
* The cloudera quickstart edition provides limited support since it is a free version. There were a lot of stability issues which had to be fixed by restarting hadoop services again and again.
* Since it was a single node hadoop cluster, performance was really bad.
