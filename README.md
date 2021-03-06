# Spark Cluster with Docker & docker-compose

## What is a spark-cluster

Spark applications run as independent sets of processes on a cluster, coordinated by the SparkContext object in your
main program (called the driver program). ... Once connected, Spark acquires executors on nodes in the cluster, which
are processes that run computations and store data for your application.

# General

A simple spark standalone cluster for your testing environment purposes. A *docker-compose up* away from you solution
for your spark development environment.

The Docker compose will create the following containers(container ip address will differ on machines):

container|Ip address
---|---
spark-master|xx.x.x.2
spark-worker-1|xx.x.x.3
spark-worker-2|xx.x.x.4
spark-worker-3|xx.x.x.5

# Inspect container IP address

**mac/linux os**

```shell
docker inspect -f '{{.Name}} - {{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' $(docker ps -aq)
```

**fish shell**

```shell
docker inspect -f '{{.Name}} - {{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' (docker ps -aq)
```

# Installation

The following steps will make you run your spark cluster's containers.

## Pre requisites

* Docker installed

* Docker compose installed

* A spark Application Jar to play with(Optional)

## Build the images

The first step to deploy the cluster will be the build of the custom images, these builds can be performed with the *
build-images.sh* script.

The executions is as simple as the following steps:

```sh
chmod +x build-images.sh
./build-images.sh
```

This will create the following docker images:

* spark-base:3.0.2: A base image based on java:alpine-jdk-8 which ships scala, python3 and spark 3.0.2

* spark-master:3.0.2: **FROM spark-base**, used to create a spark master container.

* spark-worker:3.0.2: **FROM spark-base**, used to create spark worker container(s).

## Run the docker-compose

The final step to create your test cluster will be to run the compose file:

```sh
docker-compose up --scale spark-worker=3
```

- start a single master node & 3 worker containers

## Validate your cluster

Just validate your cluster accessing the spark UI on each worker & master URL.

### Spark Master

http://localhost:9090/

![alt text](docs/spark-master.png "Spark master UI")

# Resource Allocation

This cluster is shipped with three workers and one spark master, each of these has a particular set of resource
allocation(basically RAM & cpu cores allocation).

* The default CPU cores allocation for each spark worker is 1 core.

* The default RAM for each spark-worker is 1024 MB.

* The default RAM allocation for spark executors is 256mb.

* The default RAM allocation for spark driver is 128mb

* If you wish to modify this allocations just edit the env/spark-worker.sh file.

# Binded Volumes

To make app running easier I've shipped two volume mounts described in the following chart:

Host Mount|Container Mount|Purposse
---|---|---
/mnt/spark-apps|/opt/spark-apps|Used to make available your app's jars on all workers & master
/mnt/spark-data|/opt/spark-data| Used to make available your app's data on all workers & master

This is basically a dummy DFS created from docker Volumes...(maybe not...)

# Run a sample application

Now let`s make a **wild spark submit** to validate the distributed nature of our new toy following these steps:

## Create a Scala spark app

The first thing you need to do is to make a spark application. Our spark-submit image is designed to run scala code
You can make or use your own scala app.

## Ship your jar & dependencies on the Workers and Master

A necesary step to make a **spark-submit** is to copy your application bundle into all workers, also any configuration
file or input file you need.

Luckily for us we are using docker volumes so, you just have to copy your app and configs into /mnt/spark-apps, and your
input files into /mnt/spark-files.

```bash
#Copy spark application into all workers's app folder
cp /home/workspace/crimes-app/build/libs/crimes-app.jar /mnt/spark-apps

#Copy spark application configs into all workers's app folder
cp -r /home/workspace/crimes-app/config /mnt/spark-apps

# Copy the file to be processed to all workers's data folder
cp /home/Crimes_-_2001_to_present.csv /mnt/spark-files
```

## Check the successful copy of the data and app jar (Optional)

This is not a necessary step, just if you are curious you can check if your app code and files are in place before
running the spark-submit.

```sh
# Worker 1 Validations
docker exec -ti spark-worker-1 ls -l /opt/spark-apps

docker exec -ti spark-worker-1 ls -l /opt/spark-data

# Worker 2 Validations
docker exec -ti spark-worker-2 ls -l /opt/spark-apps

docker exec -ti spark-worker-2 ls -l /opt/spark-data

# Worker 3 Validations
docker exec -ti spark-worker-3 ls -l /opt/spark-apps

docker exec -ti spark-worker-3 ls -l /opt/spark-data
```

After running one of this commands you have to see your app's jar and files.

## Use docker spark-submit

```bash
#Creating some variables to make the docker run command more readable
#App jar environment used by the spark-submit image
SPARK_APPLICATION_JAR_LOCATION="/opt/spark-apps/crimes-app.jar"
#App main class environment used by the spark-submit image
SPARK_APPLICATION_MAIN_CLASS="org.mvb.applications.CrimesApp"
#Extra submit args used by the spark-submit image
SPARK_SUBMIT_ARGS="--conf spark.executor.extraJavaOptions='-Dconfig-path=/opt/spark-apps/dev/config.conf'"

#We have to use the same network as the spark cluster(internally the image resolves spark master as spark://spark-master:7077)
docker run --network docker-spark-cluster_spark-network \
-v /mnt/spark-apps:/opt/spark-apps \
--env SPARK_APPLICATION_JAR_LOCATION=$SPARK_APPLICATION_JAR_LOCATION \
--env SPARK_APPLICATION_MAIN_CLASS=$SPARK_APPLICATION_MAIN_CLASS \
spark-submit:3.0.2

```

After running this you will see an output pretty much like this:

```bash
Running Spark using the REST application submission protocol.
2018-09-23 15:17:52 INFO  RestSubmissionClient:54 - Submitting a request to launch an application in spark://spark-master:6066.
2018-09-23 15:17:53 INFO  RestSubmissionClient:54 - Submission successfully created as driver-20180923151753-0000. Polling submission state...
2018-09-23 15:17:53 INFO  RestSubmissionClient:54 - Submitting a request for the status of submission driver-20180923151753-0000 in spark://spark-master:6066.
2018-09-23 15:17:53 INFO  RestSubmissionClient:54 - State of driver driver-20180923151753-0000 is now RUNNING.
2018-09-23 15:17:53 INFO  RestSubmissionClient:54 - Driver is running on worker worker-20180923151711-10.5.0.4-45381 at 10.5.0.4:45381.
2018-09-23 15:17:53 INFO  RestSubmissionClient:54 - Server responded with CreateSubmissionResponse:
{
  "action" : "CreateSubmissionResponse",
  "message" : "Driver successfully submitted as driver-20180923151753-0000",
  "serverSparkVersion" : "3.0.2",
  "submissionId" : "driver-20180923151753-0000",
  "success" : true
}
```

# Note

* This is intended to be used for test purposses, basically a way of running distributed spark apps on your laptop or
  desktop.
