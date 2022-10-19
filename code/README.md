# Modern Methods for Automation of Thoracic Radiography Interpretation inthe Face of Uncertainty


# Setup
## Docker Container Setup

There is a great turorial on the [CSE 6250 SunLab website](http://www.sunlab.org/teaching/cse6250/spring2021/env/env-docker-compose.html#run-exec-and-ssh-how-to-access-the-environment) on how to pull the latest docker image and setup the Docker Container which has all the functionality pre-configured. 

The docker-compose.yml file is already mostly pre-configured

You only need to link the folder on the host machine where the `CheXpert-v1.0-small` exists by updating line 32 in the docker-compose file:

 `/mnt/shared/project` 

In the same directory where the `docker-compose.yml` is stored, run:

```
docker-compose up
```

Once setup is completed and any downloads finished the Docker container is launched with the command:
```
docker-compose start && docker-compose exec bootcamp bash
```

## Hadoop setup / HDFS data load
```
# sudo su - hdfs
```
```
-bash-4.2$ hdfs dfs -mkdir -p /user/root
-bash-4.2$ hdfs dfs -chown root /user/root
-bash-4.2$ exit
```

Copy the dataset to HDFS:
```
# hdfs dfs -mkdir project
# hdfs dfs -put CheXpert-v1.0-small project
```

This is 11 GB Dataset so it could take awhile.

## Upgrade spark to a newer version

Go to your the location of your spark installation:
```
# cd $SPARK_HOME/..
```

Download Spark 2.4:
```
# wget http://www.gtlib.gatech.edu/pub/apache/spark/spark-2.4.5/spark-2.4.5-bin-hadoop2.7.tgz
```

Untar the archive:
```
# tar -xzf spark-2.4.5-bin-hadoop2.7.tgz
```

Update Environmental Variable w new version:
```
# export SPARK_HOME=/usr/lib/spark-2.4.5-bin-hadoop2.7
```

Run `spark-shell`:
```
# $SPARK_HOME/bin/spark-shell --master "local[4]" --driver-memory 8G
```

Check Spark version
```
scala> sc.version
```


## Installing BigDL Package for Spark 2.4

```
# cd /
# mkdir bigdl
# cd bigdl
# wget https://repo1.maven.org/maven2/com/intel/analytics/bigdl/dist-spark-2.4.0-scala-2.11.# 8-all/0.9.0/dist-spark-2.4.0 scala-2.11.8-all-0.9.0-dist.zip
# yum install zip -y
# yum install unzip -y
# unzip dist-spark-2.4.0-scala-2.11.8-all-0.9.0-dist.zip
# export BIGDL_HOME=/bigdl
# export BIGDL_JAR_NAME=`ls ${BIGDL_HOME}/lib/ | grep jar-with-dependencies.jar`
# export BIGDL_JAR="${BIGDL_HOME}/lib/$BIGDL_JAR_NAME"
# export BIGDL_CONF=${BIGDL_HOME}/conf/spark-bigdl.conf
# export SPARK_HOME=/usr/lib/spark-2.4.5-bin-hadoop2.7
```
# Preprocessing (Spark / HDFS)

## Image statistics

Run `image_stats.scala` in `spark-shell` with the Databricks `spark-deep-learning` package to get the Mean and Standard Deviation per Channel:
```
# cd /mnt/shared/project/CheXpert-v1.0-small
# $SPARK_HOME/bin/spark-shell --master "local[4]" --driver-memory 10G --properties-file ${BIGDL_CONF} --jars ${BIGDL_JAR}  --conf spark.driver.extraClassPath=${BIGDL_JAR} --conf spark.executor.extraClassPath=${BIGDL_JAR} -i ../code/image_statistics.scala
```
The CSV will be saved under `/mnt/shared/project/CheXpert-v1.0-small/image_statistics.csv`.


## Train/Validation/Test split

Run `train_test_split.scala` in `spark-shell` to split the images into a training and validation set used for testing:
```
# $SPARK_HOME/bin/spark-shell --master "local[4]" --driver-memory 10G -i ../code/train_val_test_split.scala
```

> NOTE: By default, this file will result in implementing a I-Zeros, U-0.66 policy. You may change this according to the comments provided in the file.  
  

We need to update Hadoop HDFS with the new training/validation sets and copy to our local storage

```
# hdfs dfs -mv project/CheXpert-v1.0-small/traintrain.csv/*.csv project/CheXpert-v1.0-small/train_train.csv
# hdfs dfs -mv project/CheXpert-v1.0-small/trainval.csv/*.csv project/CheXpert-v1.0-small/train_valid.csv
# hdfs dfs -rm -r project/CheXpert-v1.0-small/traintrain.csv
# hdfs dfs -rm -r project/CheXpert-v1.0-small/trainval.csv
# cd /mnt/shared/project/CheXpert-v1.0-small/
# hdfs dfs -get project/CheXpert-v1.0-small/train_train.csv
# hdfs dfs -get project/CheXpert-v1.0-small/train_valid.csv
```

# Training and Learning Steps using PyTorch

Run `chexpert.py`.


If you need to install Pytorch [Pytorch](https://pytorch.org/get-started/locally/)

The  `requirements.txt` file has the packages needed

Use pip package manager:
```
# pip install -r requirements.txt
```

In order to log the results from training and validation use the following command:

```
# python -u chexpert.py 2>&1 | tee chexpert.log
```
