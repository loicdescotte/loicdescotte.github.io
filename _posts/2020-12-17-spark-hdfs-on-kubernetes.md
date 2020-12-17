---
layout: post
title: Resources to run Spark and HDFS in Kubernetes
tags:
 - K8S
 - DevOps
 - Kubernetes
 - Helm
 - Spark
 - Big data
 - Scala
 - Hadoop
 - HDFS
---

Apache Spark and the Hadoop ecosystem are not really easy to install and manage manually, but using a few resources it is quite convienent to deploy an HDFS + Spark cluster on Kubernetes and submit applications on it.

## HDFS

You can use the Gradiant Helm charts provided here to install HDFS in your cluster : [https://github.com/Gradiant/charts](https://github.com/Gradiant/charts).
It is possible to override the number of datanodes and other settings using a values.yaml file, see all the defaults [here](https://github.com/Gradiant/charts/blob/master/charts/hdfs/values.yaml).

## Spark

Gradiant is also providing Spark images : [https://hub.docker.com/r/gradiant/spark](https://hub.docker.com/r/gradiant/spark)

## Submit apps

You can use a local Spark installation to run `spark-submit`, or use it directly from the Docker image as suggested in the Gradiant image [README file](https://github.com/Gradiant/dockerized-spark).

```
spark-submit \
    --master k8s://https://127.0.0.1:6443 \
    --deploy-mode cluster \
    --name hello-spark \
    --class Hello \
    --conf spark.executor.instances=2 \
    --conf spark.kubernetes.container.image.pullPolicy=IfNotPresent \
    --conf spark.kubernetes.authenticate.driver.serviceAccountName=spark \
    --conf spark.kubernetes.container.image=gradiant/spark:2.4.4 hdfs://hdfs-namenode/user/loic/jars/helloSpark.jar
```

As you can see, once the HDFS service are deployed in Kubernetes, you can use it in the cluster using the 'hdfs-namenode' service. (Use `kubectl get services` to have the list of the deployed services).  
HDFS can be reached from your Spark applications in the same way. Please note that you will need to create a Kubernetes service account with permissions to create pods and services. See the image [README file](https://github.com/Gradiant/dockerized-spark) for more details. 

To put your application jar file in HDFS, you will be able to use the httpfs service included in the HDFS Helm chart. The WebHDFS REST API documentation can be found here : [https://hadoop.apache.org/docs/r1.0.4/webhdfs.html](https://hadoop.apache.org/docs/r1.0.4/webhdfs.html).

Finally, it is possible to add custom labels to driver and executor pods, set cpu resources limits etc., using this configuration keys : [https://spark.apache.org/docs/latest/running-on-kubernetes.html#configuration](https://spark.apache.org/docs/latest/running-on-kubernetes.html#configuration).

Enjoy !
