---
layout: default
title: Install
---

# Installing Seldon

 * [Clone Seldon](#clone)
 * [Install Kubernetes](#install-kubernetes)
 * [Create Kubernetes Configuration](#configure)
     * [Persistent Storage](#storage)
     * [Seldon API Endpoint](#endpoint)
 * [Launch Seldon](#launch)
 * [Next Steps](#next-steps)
 * [Troubleshooting](#troubleshooting)

# Download Seldon<a name="clone"></a>

{% highlight bash %}
git clone https://github.com/seldonio/seldon-server -b v1.4.3
{% endhighlight %}

  * add ```seldon-server/kubernetes/bin``` to you shell PATH environment variable.

# Create a Kubernetes Cluster<a name="install-kubernetes"></a>

Seldon runs inside a [Kubernetes](http://kubernetes.io) cluster so you need to follow their [guides](http://kubernetes.io/docs) to create a cluster locally, on servers or in the cloud. We support kubernetes >= 1.2.

   * Add kubectl to your shell PATH environment variable.

If you are testing Seldon on a single machine you will need at least 6G of memory for your Kubernetes cluster. For single machine exploration we suggest using [minikube](https://github.com/kubernetes/minikube). 

To create a Kubernetes cluster on Google Cloud you can follow our [guidelines](http://docs.seldon.io/kubernetes-google-cloud.html).

# Create Kubernetes Configuration<a name="configure"></a>

Once you have a Kubernetes cluster Seldon can be started as a series of containers which run within it. As a first step you have to create the required JSON Kubernetes files. A Makefile to create these can be found in ```kubernetes/conf``` You will need to configure:

 * Persistent Storage (Kubernetes HostPath by default)
 * Seldon API Endpoint (Kubernetes NodePort by default)

## Persistent Storage<a name="storage"></a>
Seldon uses a Kubernetes [volume](http://kubernetes.io/docs/user-guide/volumes/) to store and share data between containers. The Makefile allows you to create the Kuberenetes configuration with [HostPath](http://kubernetes.io/docs/user-guide/volumes/#hostpath) or [GlusterFS](http://kubernetes.io/docs/user-guide/volumes/#glusterfs) persistent volumes. You can modify it to use other possible volumes, such as NFS, as allowed by Kubernetes.

### HostPath
To create the default HostPath kubernetes conf files set for /seldon-data do the following:

{% highlight bash %}
 cd kubernetes/conf
 make clean conf
{% endhighlight %}

   **Note** : HostPath only makes sense for demo/testing where you have a Kubernetes cluster with a single minion where all containers can share the location on the host. You will need to create the host path folder on your single kubernetes minion.

### GlusterFS
GlusterFS works well for a production setting. For this you will need to have setup your own GlusterFS cluster. We provide some [notes](glusterfs.html) to help you. Edit the Makefile and change the value of DATA_VOLUME:

{% highlight bash %}
DATA_VOLUME="glusterfs": {"endpoints": "glusterfs-cluster","path": "gv0","readOnly": false}
{% endhighlight %}

The Makefile assumes there is a volume called gs0. You will need to provide two ip addresses of two nodes in your GlusterFS cluster, e.g.:

{% highlight bash %}
 cd kubernetes/conf
 make clean conf GLUSTERFS_IP1=1.2.3.4 GLUSTERFS_IP2=1.2.3.5
{% endhighlight %}

## Seldon API Endpoint<a name="endpoint"></a>
By default the Seldon API server endpoint is set to a Kubernetes NodePort at port 30015. If you run in the cloud you can change this to LoadBalancer, e.g.

{% highlight bash %}
 cd kubernetes/conf
 make clean conf SELDON_SERVICE_TYPE=LoadBalancer
{% endhighlight %}



# Launch Seldon<a name="launch"></a>
Scripts [seldon-up](scripts.html#seldon-up) and [seldon-down](scripts.html/#seldon-down) in ```kubernetes/bin``` start and stop Seldon and should be in your PATH.

To launch seldon with all components run
{% highlight bash %}
seldon-up
{% endhighlight %}

To start with GlusterFS run 
{% highlight bash %}
SELDON_WITH_GLUSTERFS=true seldon-up
{% endhighlight %}

To shutdown seldon run
{% highlight bash %}
seldon-down
{% endhighlight %}

The first time you run [seldon-up](scripts.html#seldon-up) it may take some time to complete as it will need to download all the images from DockerHub.

On successful completion you will have a standard Seldon installation with mysql, memcache and zookeeper running within the cluster as well as a single Seldon API server and Spark cluster. The appropriate seldon-cli commands would have be run to set up the default settings and a "test" client.

# Next Steps<a name="next-steps"></a>

 * [Look at a simple content recommendation example](content-recommendation-example.html)
 * [Look at a simple prediction example](prediction-example.html)

# Troubleshooting<a name="troubleshooting"></a>

 * ***When running [seldon-up](scripts.html#seldon-up) the script waits for ever for all pods to be in running state.***

Check the reason its not finishing using: ```kubectl get all``` and ```kubectl get events```

 * If you find there are "ErrImagePull" states for some Pods check the nodes have access to the internet.

 * If you find that images are in "Pending" state then it may be due to a lack of resources on your Kubernetes cluster.

If you plan to test Seldon on a non-local cluster you will need to ensure your cluster is large enough to run all the Seldon services or disable the Kubernetes **LimitRanger** plugin. In the current version of Kubernetes to disable this plugin do the following. Edit ```<kubernetes>/cluster/<provider>/config-default.sh``` and remove LimitRanger from the following line:
{% highlight bash %}
ADMISSION_CONTROL=NamespaceLifecycle,LimitRanger,SecurityContextDeny,ServiceAccount,ResourceQuota,PersistentVolumeLabel
{% endhighlight %}

 * ***Testing on a single node or laptop things are running very slowly.***

Check you have enough memory. At least 6G is needed to run everything locally on a single node. If you are using minikube then you can start a minikube kubernetes with 6G of memory with ```minikube start --memory=6000```

In addition, the following may help:

1. Reduce the memory allocation for ***mysql*** and ***seldon-server*** pods before running [seldon-up](scripts.html#seldon-up) using the following commands:

{% highlight bash %}
 cd kubernetes/conf
 make clean conf MYSQL_RESOURCES='"requests":{ "memory" : "2Gi" }' SELDON_SERVER_RESOURCES='"requests":{ "memory" : "2Gi" }'
{% endhighlight %}

This command reduces the memory allocation for the ***mysql*** and ***seldon-server*** pods from the default 3GB each to 2GB each. If you just need to run the Reuters recommendation and Iris prediction examples, even 1 GB for ***mysql*** will work.

2. Disable Spark, in case you do not intend to use it. To do so, invoke [seldon-up](scripts.html#seldon-up) like this:
{% highlight bash %}
SELDON_WITH_SPARK=false seldon-up
{% endhighlight %}

If you are using a Vagrant VM to run your kubernetes cluster ensure it has 6G of memory available from the host machine.

 * ***Pods are going into CrashLoopBackoff***

Check you have enough memory. At least 6G is needed to run everything locally on a single node. If you are using minikube then you can start a minikube kubernetes with 6G of memory with ```minikube start --memory=6000```

 * ***Its taking a long time to start after running [seldon-up](scripts.html#seldon-up) for the first time***

The first time you run seldon-up it will need to pull all the container images from Docker Hub. This may take some time on a slow internet connection.

If you are using minikube, first remove old nodes using the ```minikube delete``` command.
