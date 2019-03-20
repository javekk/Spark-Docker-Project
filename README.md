# Spark-Docker-Project
Report in which is explained some steps to run spark project on a cluster with docker.



### Introduction

In this project  several way to **run** and **deploy** the code for music recommendations [[1]](https://github.com/sryza/aas.git),[[2]]( https://github.com/sryza/aas/tree/master/ch03-recommender)  have been examined with the respective (Trimmed) dataset[3] published by Audioscrobbler. 

Staring from run and deploy the recommender on local machine using IDE such as Intellij, and spark-submit on the console, then going for deployment on **Docker** and after that using **docker-compose** to deploy on several nodes(1 master + 2 Slaves), finally store data in **HDFS**.



### Deploy on Docker

In order to build to the image it was used the **Dockerfile** provided by *P7h* on Github[[5]]( https://storage.googleapis.com/aas-data-sets/profiledata_06-May-2005.tar.gz), with the following command:

```bash
$ docker build -t p7hb/docker-spark .
```

Then it was necessary to pull the image using the following command:

```bash
$ docker pull p7hb/docker-spark
```

At that point, before run the docker **container**, some changes have been needed. In order to facilitate the operations the whole dataset has been moved in the same directory of the `.jar` file. Then since it should have been run on a docker container the path in the code was change with the docker container path `/home`.  The following step was to launch the Docker container with the following command:

```bash
$ docker run -it -p 4040:4040 -p 8082:8080 -p 8081:8081 -v <dataset_path>:/home -h spark --name=spark p7hb/docker-spark
```

A further explanation of this command is needed, `-it` is used to allocate a tty for the container process, while the option `-p` binds the container's internal ports(on the right) with the host port(on the left). `-v` is specific for the **volume**, with this command it is possible to *"mount"* the directory dataset, in which the dataset and the .jar are contained, in the `/home` directory within the docker container's file system. The  `-h` command sets the *hostname*, `--name` identifies the containers itself and finally for `docker run` command it is necessary to specify the image to derive the container from(`p7hb/docker-spark`).

After this command has been run it was possible to effectively run the recommender within the container using the `spark-submit` command:

```bash
$ <spark_path>/bin/spark-submit --class eu.upm.bd.RunRecommender --master local[*] /home/rec_bd_project_2.jar 1000072
```



### Deploy on several nodes

This part was divided in two steps, first only 1 master and 1 slave were used than in order to slightly scale up 1 master and 2 slaves. To develop the cluster mode it has been used the Docker build for Apache Spark provided by *gettyimages* on Github[[6]](https://github.com/gettyimages/docker-spark).

##### 1 master + 1 slave

In the very first step of this part it was necessary to modify the `.yml` file, with the right version of *spark*; this file contains all the settings for both master and worker.

Then again for make it easy the dataset and the `.jar` have been placed in the *data* folder. At this step it was possible to run the following command from the root to run the cluster:

```bash
$ docker-compose up 
```

that builds, (re)creates, starts, and attaches to containers for a service[[7]]( https://docs.docker.com/compose/reference/up/). 

And here it is possible to see on the spark's GUI on *`localhost:8080`*.

Then in another terminal it was used the `docker exec` command as follow:

```bash
$ docker exec -it dockerspark_master_1 bash
```

to open the master's bash (*dockerspark_master_1* is the name of the master). From that bash it was possible to run the recommender in a cluster mode using the following command:

```bash
$ ./bin/spark-submit --class eu.upm.bd.RunRecommender --master spark://172.17.0.1:6066 --deploy-mode cluster <jar_path>/rec.jar 1000072
```

After this command it was possible to see the output on the spark browser GUI, going to the specific worker IP.

##### 1 master + 2 slaves

To scale up with more than one slave, two in this case, it was possible to used the same command as the previous step but change the number of containers for the worker service to 2, using the following command after the `up` command:

```bash
$ docker-compose scale worker=2
```

### Store data in HDFS:Hadoop

Two new files were needed in order to deploy **HDFS** inside docker containers. First a new `.yml` file from *big_data_europe* on github[[8]](https://github.com/big-data-europe/docker-hadoop-spark-workbench/blob/master/docker-compose.yml), because two new services were needed, **datanode** and **namenode**. Then the configuration file for Hadoop from the same repository[[9]](https://github.com/big-data-europe/docker-hadoop-spark-workbench/blob/master/hadoop.env).

After the modification of the `.yml` and again the modification of the path in the code with the hdfs FS it was possible to run the `up` command in the root.

```bash
$ docker-compose up
```

Done this it was necessary to effectively add the dataset on the *datanode*, in this way,(1) run `exec` on the Datanode image:

```bash
$ docker exec -it docker_datanode_1 bash
```

(2)within the datanode bash, create a new directory:

```bash
$ hadoop fs -mkdir -p /user/root
```

(3)move the data on hdfs within the container:

```bash
$ hadoop fs -put <files_path>/hadoop/dfs/data /user/root/data
```

here a screen-shot of the dataset see from browser GUI on this link `http://172.17.0.1:8088/filebrowser/`:

Finally it was possible to run the Recommender in client mode with as follow:

```bash
$ docker exec -it spark-master bash
```

```bash
$ <spark_path>/spark/bin/spark-submit --class eu.upm.bd.RunRecommender --master local[*] <jar_path>/rec.jar 1000072
```

### References

[1] https://github.com/sryza/aas.git, "Advanced Analytics with Spark: patterns for learning from data at scale”, Sandy Ryza, Uri Laserson, Sean Owen, Josh Wills. O’Reilly"

[2] https://github.com/sryza/aas/tree/master/ch03-recommender, "Chapter 3: Recommender"

[3] https://storage.googleapis.com/aas-data-sets/profiledata_06-May-2005.tar.gz ,"Audiocrobbler dataset"

[4] https://www.scala-sbt.org/, "sbt website"

[5] https://github.com/P7h/docker-spark, "docker-spark for dockerfile"

[6] https://github.com/gettyimages/docker-spark, "Docker build for apache spark"

[7] https://docs.docker.com/compose/reference/up/, "Docker-compose up"

[8] https://github.com/big-data-europe/docker-hadoop-spark-workbench/blob/master/docker-compose.yml, "yml for hadoop"

[9] https://github.com/big-data-europe/docker-hadoop-spark-workbench/blob/master/hadoop.env, "yml for hadoop"

