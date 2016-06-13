---
layout: default
title: Benchmarking guide
---

# Seldon Benchmarking Guide
This guide will go through detailed steps to show how a Seldon setup can be benchmarked. In this case we will:

 * Use Amazon Web Services.
  * Create a cluster with 3 m3.large AWS instances
  * Run Seldon with default settings and no extra optimization
 * Run tests on the Movielens 10 Million dataset with the item-similarity algorithm for content recommendation.
 * Run a loadtest at 50 recommendation API calls per second.

## Create Kubernetes cluster on AWS

 * [Create a GlusterFS cluster in an AWS VPC](glusterfs.html).
 * [Create a Kubernetes AWS cluster](http://kubernetes.io/docs/getting-started-guides/aws/). The configuration used is shown below. You will need to change as appropriate with desired key, secret, region and bucket name.
{% highlight bash %}
   export AWS_ACCESS_KEY_ID=<YOUR_ACCESS_KEY>
   export AWS_SECRET_ACCESS_KEY=<YOUR_SECRET_KEY>
   export AWS_DEFAULT_REGION=eu-west-1
   export KUBE_AWS_ZONE=eu-west-1c
   export KUBE_AWS_INSTANCE_PREFIX=k8s-loadtest
   export MASTER_SIZE=m3.large
   export NODE_SIZE=m3.large
   export NUM_NODES=4
   export AWS_S3_REGION=eu-west-1
   export AWS_S3_BUCKET=<your S3 bucket>
   export KUBERNETES_PROVIDER=aws
{% endhighlight %}
 * Update Seldon Kubernetes configuration
    * Update ```seldon-server/kubernetes/conf/Makefile```, setting the glusterfs ip addresses as appropriate for your glusterfs configuration.
{% highlight bash %}
   DATA_VOLUME="glusterfs": {"endpoints": "glusterfs-cluster","path": "gv0","readOnly": false}
   SELDON_SERVICE_TYPE=LoadBalancer
   GLUSTERFS_IP1=192.168.0.149
   GLUSTERFS_IP2=192.168.0.248
{% endhighlight %}
 * Modify the file ```seldon-server/kubernetes/conf/server.json.in``` to provide 2 API servers, update to:
{% highlight json %}
 "replicas": 2,
{% endhighlight %}
   Create the conf with ```make clean conf```

 * Label one node as iago loadtest node. We will use this node to run the loadtester later. Your node name will differ.
{% highlight bash %}
   kubectl label nodes ip-172-20-0-71.eu-west-1.compute.internal role=iago
{% endhighlight %}
 * Cordon off iago loadtest node so its not used for Seldon. 
{% highlight bash %}
   kubectl cordon ip-172-20-0-71.eu-west-1.compute.internal
{% endhighlight %}
 * Start Seldon
{% highlight bash %}
   SELDON_WITH_GLUSTERFS=true seldon-up.sh
{% endhighlight %}

## Create item-similarity model for Movielens 10 Million
   [Run the Movielens 10 Million training and check the API is providing recommendations](ml10m.html).

## Run Iago Load Test 
  * Create iago replay script. Details of the the scripts available to create iago replay scripts are described [here](iago.html). In ths case we will create a replay script using 10,000 random users with 50,000 recommendation (and action) API calls using seldon-server as the endpoint.
{% highlight bash %}
   cd seldon-server/docker/iago
   ./create_recommendation_replay.sh ml10m /seldon-data/loadtests/example.ml10m.replay.txt 10000 50000 seldon-server
{% endhighlight %}
* Upload to Kubernetes master, mount glusterfs and place in /seldon-data/loadtest, the steps are illustrated below:
{% highlight bash %}
   ssh <kubernetes master>
   apt-get install glusterfs-client
   mkdir /mnt/glusterfs
   mount.glusterfs 192.168.0.149:/gv0 /mnt/glusterfs
   sftp <replay-file> /mnt/glusterfs/loadtest
{% endhighlight %}
  * Uncordon the iago node so we can start the iago loadtest service in it.
{% highlight bash %}
   kubectl uncordon ip-172-20-0-71.eu-west-1.compute.internal
{% endhighlight %}
  * Start iago service. It will use the NodeSelector to run on the iago node.
{% highlight bash %}
   cd seldon-server/kubernetes/conf/dev
   kubectl create -f iago.json
{% endhighlight %}
  * Log into iago container
{% highlight bash %}
   kubectl exec -ti `kubectl get pods | grep iago | cut -d' ' -f1` -- /bin/bash
{% endhighlight %}
  * Start load test at 100 requests per second. As the replay script contains /js/action and /js/recommendation API calls as would be the case in a production scenario this corresponds to 50 recommendation calls per second.
{% highlight bash %}
  ./start-load-test.sh /seldon-data/loadtest/example.ml10m.replay.txt 100
{% endhighlight %}

## Results
You can view iago stats on its UI. A loadbalancer would have been created. The url for latency is of the form below, replace with your loadbalancer external DNS name:
{% highlight bash %}
http://<iago-ui-loadbalncer>:9994/graph/?g=metric:client/request_latency_ms
{% endhighlight %}

After some peaks at the start the 75% percentile settles down to below 100ms, 90% percentile to below 250ms, and the 95% to below 500ms.

![loadtest 95% percentile](/img/ml10m-loadtest-1.png)

Looking at the 99% percentile shows there are some outliers which would suggest further optimization and increased infrastructure size would be necessary to get this within acceptable limits.

![loadtest 99% percentile](/img/ml10m-loadtest-2.png)

## Further Optimization
For this benchmark item-similarity is mainly using the Mysql DB to get the similarities created from the Spark modelling job so further optimization should focus on Mysql read optimization and the front end servers cpu and memory. In general to get further decreases in latency and to handle higher loads the further optimizations shown below could be investigated:
 
 * Increase AWS instance sizes to provide more CPU and memory.
 * Increase the number and memory settings for the Seldon API servers. 
   * The seldon-server docker image can be rebuilt with changed Apache Tomcat settings. The Tomcat settings are in ```seldon-server/docker/seldon-server/scripts/start-server.sh```. 
   * The resource request for the server Pod can be changed in ```seldon-server/kubernetes/conf/Makefile```:
{% highlight bash %}
   SELDON_SERVER_RESOURCES="requests":{ "memory" : "3Gi" }
{% endhighlight %}
 * Increase the Mysql memory and optimize Innodb settings.
   * The mysql docker container is in ```seldon-server/docker/mysql```. The my.cnf file can be modified and the image rebuilt for your own use case.
   * The Kubernetes conf Makefile ```seldon-server/kubernetes/conf/Makefile``` has resource setting for mysql which can be increased, change:
{% highlight bash %}
   MYSQL_RESOURCES="requests":{ "memory" : "5Gi" }
{% endhighlight %}
 * Run Mysql on SSD disks.
 * Run Mysql with replication to spread database load over several read replicas.
 * Increase Memcache size and CPU. Change the memcache.json.in file in ```seldon-server/kubernetes/conf/``` and modify ```seldon-server/kubernetes/conf/Makefile``` by changing the resource request:
{% highlight bash %}
   MEMCACHE_RESOURCES="requests":{ "memory" : "550Mi" }
{% endhighlight %}


