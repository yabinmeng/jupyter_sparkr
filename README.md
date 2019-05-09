# Overview

This document describes the procedure of how to use [Jupyter notebook](https://jupyter.org/) as a front-end data science analytical tool to examine data stored in DSE (core/C*) through DSE Analytics (Spark) and R. 

## Environment Setup

In order to better describe the proceure, I use an example in a testing environment that includes:

* **One DSE (6.7.3) cluster with DSE Analytics (Spark) enabled** 

In this document, I'll skip the procedure of provisioning and configuring a DSE cluster. Please check official DSE document for more information. For simplicity purpose, the testing DSE cluster is a single-DC, 2-node cluster. 

* **One Jupyter client node/instance**

The focus of this document is on how to set up and prepare this client node/instance so end users can create a Jupyter notebook (from its web UI) and use it for advanced data science oriented analysis on the C* data that is stored in the DSE cluster, via Spark and R.

The client node/instance needs to be able to successfully connect to the DSE cluster via native CQL protocol (default port 9042). 

# Data Processing with R and DSE

## Native DSE support for R

DSE has native support for using R via SparkR through a console. On any DSE Anatlycis node, install R first and then execute the following command will bring up the SparkR console. 
```
  $ dse sparkR
```

If the above command is executed successfuly, we should see something similar to the output below, which tells a few important information:
* The log file location
* Spark and R versions
* The SparkSession (named as 'spark') that can be used in the console directly, which can be used to fetch data from C* tables.

From the console command input (after '>'), you can type in the required R code.

```
  The log file is at /home/automaton/.sparkR.log

  R version 3.4.4 (2018-03-15) -- "Someone to Lean On"
  Copyright (C) 2018 The R Foundation for Statistical Computing
  Platform: x86_64-pc-linux-gnu (64-bit)

  ... ...

  Launching java with spark-submit command /usr/share/dse/spark/bin/spark-submit   "sparkr-shell" /tmp/Rtmp8VUJku/backend_port56551bb88aee

   Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /___/ .__/\_,_/_/ /_/\_\   version  2.2.3.4
      /_/


   SparkSession available as 'spark'.
  During startup - Warning message:
  In SparkR::sparkR.session() :
    Version mismatch between Spark JVM and SparkR package. JVM version was 2.2.3.4 , while R package version was 2.2.2
  > 
```

This method, although works, has a major limitation: It requires granting the end user proper access privilege to the DSE Analytics nodes. For many cases, it is treated as a security violation by granting direct end user access to production nodes; and therefore forbidden by many organization's IT security/compliance department.


## Integrate DSE R Support with Jupyter Notebook

In order to address the above concerns, we can set up a Juypter notebook (server) on a dedicated client node/instance that can connect to the DSE cluster as a regular client. The procedure of doing so is described in details below. 

**NOTE**: the installation commands listed below are all based on Ubuntu 16.04.3 LTS OS. For other OS, please change accordingly.


### Install DSE Binaries

It is needed to install DSE binaries on the client node so Jupyter R kernel can pick up the right DSE libraries to utilize DSE SparkR funcationality. You can use any supported installation method (linux package, tarball, etc.) to install DSE binaries on the client node. But please make sure do NOT start DSE service because all we need here are DSE libraries. In my test, I use linux ATP package installation method that can be found at:
https://docs.datastax.com/en/install/6.7/install/installDEBdse.html

### Install Jupyter

Run the following commands to install Jupyter (server)
```
  $ sudo pip install --upgrade pip
  $ sudo python -m pip install jupyter
```

Verify the installed version by the following command:
```
  $ jupyter --version
```

### Install R
The commands to install R on the node instance is as below:
```
  $ sudo sh -c 'echo "deb http://cran.rstudio.com/bin/linux/ubuntu trusty/" >> /etc/apt/sources.list'
  $ gpg --keyserver keyserver.ubuntu.com --recv-key E084DAB9 && gpg -a --export E084DAB9
  $ sudo apt-key add -
  $ sudo apt-get update 
  $ sudo apt-get install r-base
```

To verify the installation, you can run the following command to check the installed R version:
```
  $ R --version
```

### Install Jupypter R Kernel and other R Libraries
In oder to use R within Juypter, we need to install [IRkernel](https://github.com/IRkernel/IRkernel). The procedure of doing so is as below:

First, bring up the R console with "sudo" privilege:
```
  $ sudo R
```

Second, from the R console command line, run the following commands. The installation process may take a while to complete
```
  > install.packages('IRkernel')
  > IRkernel::installspec(user = FALSE)
```

Third (optionally), in order to use more extended R libraries, we need to install them as well, following the same approach. For example, in my test, I installed [dplyr](https://dplyr.tidyverse.org/) and [ggplot2](https://ggplot2.tidyverse.org/) for more advanced R data manipulation and graph processing. The commands are as below:
```
  > install.packages('dplyr')
  > install.packages('ggplot2')
```

### Configure Remote DSE Cluster Connection for SparkR
By default, DSE SparkR library tries to connect to the local node as the Spark master node. For this testing, since the Spark master node (one of the DSE Analytics node) is on a remote cluster, we need to tell SparkR where the remote Spark master node is. This is done by making the follwing configuration changes in **spark-defaults.conf** of the DSE Spark installation (e.g. /etc/dse/spark/spark-defaults.conf). We can also make other Spark related settings here. For example, the maximum number of CPUs and memory allocated to the SparkSession used by SparkR. 

```
  spark.master  dse://<DSE_Analytic_Node_IP>:9042
  spark.cassandra.connection.host  <DSE_Analytic_Node_IP>
  spark.hadoop.cassandra.host  <DSE_Analytic_Node_IP>
  spark.hadoop.fs.defaultFS  dsefs://<DSE_Analytic_Node_IP>
  spark.cores.max  1
  spark.executor.memory  2G
```

Please **note** that:
1. For DSE Analytics (Spark), the format of the Spark master node IP address is as: ***dse://<DSE_Analytic_Node_IP>:9042*** and it can be the IP of any DSE Analytics node in the cluster. It is not necessarily to use the "real" Spark master IP. DSE is able to route the connection to the right Spark master internally.

2. It is important to limit the CPU and memory usage by the SparkR session. Otherwise, it will use all available CPU and memories allocated to Spark worker/executor on DSE servers and my block other Spark applications indefinitely. 

### Start Jupyter Server
At this point, the Juypter server is ready to start, with the support for R and integration with Spark (through SparkR). Run the following command to start the Jupyter server.
```
  $ dse exec jupyter notebook --ip=0.0.0.0 --port=9090
```

The above command starts a Jupyter server process that listens on port **9090** on all IP address **0.0.0.0**. This is needed if we want to access the Jupyter notebook web UI from a public IP. From the web UI, you can choose to create a R notebook. An example screenshot is as below:

<img src="https://github.com/yabinmeng/jupyter_sparkr/blob/master/resources/jupyter_r_home.png" alt="Jupyter Web UI]" width="600">

When a new R notebook is created, it will start a (Jupyter) R kernel which is able to run Spark code via SparkR. If the R kernel is successfully started, you should see something similar to be low from Jupyter server log.

```
[I 04:17:04.947 NotebookApp] Kernel started: 0e76461d-c61c-4714-b1d5-d8cac1e88fa7
Spark package found in SPARK_HOME: /usr/share/dse/spark
Launching java with spark-submit command /usr/share/dse/spark/bin/spark-submit   sparkr-shell /tmp/Rtmp95LjKZ/backend_port614854193131
WARN  2019-05-09 04:17:11,463 org.apache.spark.util.Utils: Service 'SparkUI' could not bind on port 4040. Attempting port 4041.

 Welcome to
    ____              __
   / __/__  ___ _____/ /__
  _\ \/ _ \/ _ `/ __/  '_/
 /___/ .__/\_,_/_/ /_/\_\   version  2.2.3.4
    /_/


 SparkSession available as 'spark'.
During startup - Warning message:
In SparkR::sparkR.session() :
  Version mismatch between Spark JVM and SparkR package. JVM version was 2.2.3.4 , while R package version was 2.2.2
[I 04:17:16.118 NotebookApp] Adapting to protocol v5.0 for kernel 0e76461d-c61c-4714-b1d5-d8cac1e88fa7
[I 04:17:29.489 NotebookApp] Saving file at /MyDSESparkR.ipynb
```

Meanwhile, from Spark master web UI, you should see a running application named under "sparkR", as below. 

<img src="https://github.com/yabinmeng/jupyter_sparkr/blob/master/resources/jupypter_sparkr_master.png" alt="Jupyter Web UI]" width="600">

#### Limit SparkR Resource Usage

Please also **notice** the CPU and memory usage as limiteded per our previous settings. This is very important to set the limits because the SparkR session launched from the notebook is a long running session. Unless the notebook user explicitly terminate the R kernel (see below), the SparkR session will stay alive until Jupyter server restarts. If we don't set the Spark resource usage limit, the SparkR session will grab all available resources, leading all future Spark applications stuck waiting. 

##### Terminate Jupyter R Kernel Properly

In order to properly clean up the resources being allocated to the SparkR session at the Spark cluster, the notebook user needs to explicitly terminate the R kernel associated with it from the notebook UI, as below.

<img src="https://github.com/yabinmeng/jupyter_sparkr/blob/master/resources/jupyter_rkernel_shutdown.png" alt="Jupyter Web UI]" width="600">

Please **note** that simply  closing the notebook web browser window doesn't terminate the R kernel automatically. The SparkR session still stays alive and is buffered by Jupyter server for future usage. Next time when a new notebook is created, it will reuse the buffered SparkSession (if any) without creating a new session.

# R Notebook with SparkR Demo 

The following R notebook gives a demo on how to use the SparkR session to fetch data from a DSE C* table and then use R code for some statistical analysis. 

![R Notebook with Spark](https://github.com/yabinmeng/jupyter_sparkr/blob/master/resources/notebook/MyDSESparkR.ipynb)
