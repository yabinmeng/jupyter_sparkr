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

This method, although works, has a few practical limitations.

First, since it is console based, it is literally a single-user environment. Different users trying to do concurrent R data analysis on the same DSE node need to launch their own consoles, which is resource heavy.

Second and more importantly, it requires granting the end user proper access privilege to the DSE Analytics nodes. For most cases, it is treated as a security violation by granting direct end user access to production nodes; and therefore forbidden by many organization's IT security/compliance department.


## Integrate DSE R Support with Jupyter Notebook

In order to address the above concerns, we can set up a Juypter notebook (server) on a dedicated client node/instance that can connect to the DSE cluster as a regular client. The procedure of doing so is described in details below:

### Install DSE Binaries

It is needed to install DSE binaries on the client node so Jupyter R kernel can pick up the right DSE libraries for utilizing DSE SparkR funcationality. You can use any supported installation method (linux package, tarball, etc.) to install DSE binaries on the client node. But please make sure do NOT start DSE service because all we need here is proper DSE libraries. In my test, I use linux ATP package installation method that can be found at:
https://docs.datastax.com/en/install/6.7/install/installDEBdse.html


### Install R


