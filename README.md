# Building a Recommender with Apache Mahout on Amazon Elastic MapReduce (EMR)

### Start the EMR Cluster
Signup to EMR and need to start EMR cluster with Mahout


### Exact sample data set and convert
In this scenario, I exact Get the sample MovieLens data set.

```
wget http://files.grouplens.org/datasets/movielens/ml-1m.zip
unzip ml-1m.zip
```

Convert ratings.dat, trade “::” for “,”, and take only the first three columns:

```
cat ml-1m/ratings.dat | sed 's/::/,/g' | cut -f1-3 -d, > ratings.csv
```

Put ratings file into HDFS:

```
hadoop fs -put ratings.csv /ratings.csv
```

### Run the recommender

and run the recomonder job as below

```
mahout recommenditembased --input /ratings.csv --output recommendations --numRecommendations 10 --outputPathForSimilarityMatrix similarity-matrix --similarityClassname SIMILARITY_COSINE
```

Look for the results in the part-files containing the recommendations:


```
 hadoop fs -ls recommendations
```

and output as below
```
Found 4 items
-rw-r--r--   1 hadoop hadoop          0 2021-03-20 12:01 recommendations/_SUCCESS
-rw-r--r--   1 hadoop hadoop     196949 2021-03-20 12:01 recommendations/part-r-00000
-rw-r--r--   1 hadoop hadoop     197668 2021-03-20 12:01 recommendations/part-r-00001
-rw-r--r--   1 hadoop hadoop     197537 2021-03-20 12:01 recommendations/part-r-00002
```


```
hadoop fs -cat recommendations/part-r-00000 | head
```

```
3       [2376:5.0,2243:5.0,2110:5.0,2968:5.0,2374:5.0,2375:5.0,594:5.0,2111:5.0,3035:5.0,3168:5.0]
6       [1187:5.0,3034:5.0,594:5.0,2572:5.0,3035:5.0,2110:5.0,3298:5.0,3826:5.0,3100:5.0,3299:5.0]
9       [2112:5.0,196:5.0,1517:5.0,1912:5.0,1320:5.0,1449:5.0,198:5.0,3100:5.0,265:5.0,1120:5.0]
12      [3363:5.0,1378:5.0,930:5.0,3039:5.0,922:5.0,3361:5.0,2640:5.0,594:5.0,928:5.0,2054:5.0]
15      [170:4.5212846,3948:4.4861436,3793:4.473397,3861:4.4585075,3512:4.4527225,3354:4.4498577,3566:4.4497514,3484:4.4416394,3615:4.43577,3316:4.4193387]
18      [2112:5.0,1320:5.0,2110:5.0,1584:5.0,2376:5.0,2243:5.0,3430:5.0,3035:5.0,1187:5.0,3168:5.0]
21      [3535:5.0,2600:5.0,1356:5.0,3752:5.0,3317:5.0,3160:5.0,3785:5.0,3617:5.0,1909:5.0,3863:5.0]
24      [2374:5.0,3168:5.0,2110:5.0,529:5.0,2375:5.0,2243:5.0,3035:5.0,1188:5.0,265:5.0,2641:5.0]
27      [2110:5.0,1912:5.0,594:5.0,1188:5.0,2968:5.0,3629:5.0,198:5.0,1253:5.0,3035:5.0,2243:5.0]
30      [1231:5.0,2624:5.0,2161:5.0,2470:5.0,1957:5.0,431:5.0,1892:5.0,1994:5.0,3002:5.0,1263:5.0]
cat: Unable to write to output stream.
```

## Bulding a service

Next, we’ll use this lookup file in a simple web service that returns movie recommendations for any given user.

1) Get Twisted, and Klein and Redis modules for Python.

```
sudo pip3 install twisted
sudo pip3 install klein
sudo pip3 install redis
```

2) Install Redis and start up the serve

```
wget http://download.redis.io/releases/redis-2.8.7.tar.gz
tar xzf redis-2.8.7.tar.gz
cd redis-2.8.7
make
./src/redis-server &
```

3) Build a web service that pulls the recommendations into Redis and responds to queries. Put the following into a file, e.g., “hello.py”

```
from klein import run, route
import redis
import os

# Start up a Redis instance
r = redis.StrictRedis(host='localhost', port=6379, db=0)

# Pull out all the recommendations from HDFS
p = os.popen("hadoop fs -cat recommendations/part*")

# Load the recommendations into Redis
for i in p:

  # Split recommendations into key of user id 
  # and value of recommendations
  # E.g., 35^I[2067:5.0,17:5.0,1041:5.0,2068:5.0,2087:5.0,
  #       1036:5.0,900:5.0,1:5.0,081:5.0,3135:5.0]$
  k,v = i.split('t')

  # Put key, value into Redis
  r.set(k,v)

# Establish an endpoint that takes in user id in the path
@route('/<string:id>')

def recs(request, id):
  # Get recommendations for this user
  v = r.get(id)
  return 'The recommendations for user '+id+' are '+v


# Make a default endpoint
@route('/')

def home(request):
  return 'Please add a user id to the URL, e.g. http://localhost:8086/1234n'

# Start up a listener on port 8086
run("localhost", 8086)
   
```

4) run the python file

```
twistd -noy hello.py &
```

5) Test the web service with user id "34":

```
curl localhost:8080/34
```
that output as below

```
[7:5.0,2088:5.0,2080:5.0,1043:5.0,3107:5.0,2087:5.0,2078:5.0,3108:5.0,1042:5.0,1028:5.0]
```


##### IMPORTANT - finaly needs to shut down the cluster before exit
