# cloud-hadoop-benchmarks

Use common Hadoop benchmarks to evaluate the performance of various cloud provider platforms.

## Overview

Cloudbreak simplifies the deployment of Hortonworks platforms in cloud environments such as Amazon Web Services, Microsoft Azure and Google Cloud Platform. Cloudbreak enables the enterprise to quickly run big data workloads in the cloud while optimizing the use of cloud resources.

Cloudbreak is also a great tool for automating the deployment of multiple clusters across different cloud platform providers.  This ensures our ability to deploy known HDP configurations while minimizing the vartions to just instance and/or storage types.

There are a number of common benchmarking tools avaialble for Hadoop.  I will use TestDFISIO, TeraGen, TeraSort and TPC-DS.

## Prerequisites

### Cloudbreak

I recommend using Cloudbreak to deploy clusters using the provided blueprints and cluster definitions to achieve similar results to my testing.  However it's not required.

- Cloudbreak 2.4.1
- Ambari 2.6.1.3
- HDP 2.6.4.5-2

You can create a local instance of Cloudbreak using Vagrant and Virtualbox.  I have a tutorial that covers how to do that: [Using Vagrant and Virtualbox to create a local instance of Cloudbreak 2.4.1](https://community.hortonworks.com/articles/194076/using-vagrant-and-virtualbox-to-create-a-local-ins.html).

If you follow my tutorial above, you can place all of the json files in this repo inside of the vagrant directory for your Cloudbreak deployer.  They will be accessible in the `/vagrant` directory making it easy to use them with the Cloudbreak CLI.

### Cloudbreak CLI

You should install the Cloudbreak CLI onto your Cloudbreak deployer node.  This will make automating the benchmarking much easier rather than using the UI Wizards.  The documentation walks you through that: [Installing Cloudbreak CLI](https://docs.hortonworks.com/HDPDocuments/Cloudbreak/Cloudbreak-2.4.2/content/cli-install/index.html).  Before using the CLI, you should regiser your Cloudbreak username and password with the CLI using the following command:

`cb configure --server <cloudbreak hostname> --username <cloudbreak username> --password <cloudbreak password>`

### Cloud Credentials

You need credentials for your cloud provider platform of choice.  You need to define these credentials within Cloudbreak.

## Running the benchmarks

You should log into the client_1 node created by Cloudbreak using the ssh key you specified in the cluster definition.  All tests should be performed on the client node.

### Cluster Definitions

The cluster definitions included in this repo were generated with Cloudbreak using the Create Cluster wizard.  You can choose to recreate your own cluster definitions or you can use the attached cluster JSON files.  These files are used with the Cloudbreak CLI.

You should modify the cluster defintion json files to meet your needs.  Common sections that will need to be changed include:

- placement
  - availabilityZone - You may want to use a differnet availability zone
  - region - You may want to use a different region
- general
  - credentialName - This should correspond to the credential name for your cloud provider
- cluster
  - password - You should change the Ambari admin password
- instanceGroups
  - instanceType - You may want to use different instance types
  - volumetype - You may want to use different storage types
  - volumeCount - You may want to use a different number of volumes
  - volumeSize - You may want to use a different size of volume
  - securityGroupId - This should correspond to an existing security group you have created for your cloud provider
- network
  - subnetId - This should correspond to an existing subnet you have created for your cloud provider
  - vpcId - This should correspond to an existing subnet you have created for your cloud provider

### Blueprints

I recommend using the provided blueprints for benchmark purposes.  However, you may want to perform A-B testing with small changes the cluster layout.  I have provided 2 hive-focused blueprints.  To register the blueprints, you need to be on your Cloudbreak deployer node and run the following:

cb blueprint create from-file --file hive-tpcds-blueprint.json --name hive-tpcds

### Create Cluster

You can chose to use the Cloudbreak UI wizard to create clusters.  However, it's much faster to use the CLI.  To create a cluster using the CLI, run the following command:

`cb cluster create --cli-input-json <cluster definition json file> --name <cluster name>`

For example:

`cb cluster create --cli-input-json hive-tpcds-cluster-m4-xlarge.json --name myoung-hive-tpcds-2`

### Login to Client Node

Once the cluster is running, you should log into the client node.  The client node will be on the cluster details page in Cloudbreak.  Here is an exmaple:

![Image](https://github.com/Jaraxal/hive-testbench/blob/master/cluster-details.png)

You should use the "cloudbreak" username and the key you specified in the cluster defintion.  For example:

`ssh -i ~/Downloads/cloudbreak-crashcourse.pem  cloudbreak@13.57.199.34`

Before running the benchmarks, you should create the required HDFS directories.

```
sudo -u hdfs hadoop fs -mkdir /user/cloudbreak
sudo -u hdfs hadoop fs -chown cloudbreak /user/cloudbreak
sudo -u hdfs hadoop fs -mkdir /benchmarks
sudo -u hdfs hadoop fs -chown cloudbreak /benchmarks
```

### TestDFSIO

TestDFSIO has a `write` and a `read` component.  You should test both of these.

***NOTE: Paths specified are specific to HDP 2.6.4.5.  If you use a different version of HDP, you should update the paths accordingly.

#### Test 50GB

To test `write` for 50GB:

```
time hadoop jar /usr/hdp/2.6.4.5-2/hadoop-mapreduce/hadoop-mapreduce-client-jobclient-2.7.3.2.6.4.5-2-tests.jar TestDFSIO -D mapred.output.compress=false -write -nrFiles 50 -fileSize 1000
```

To test `read` for 50GB:

```
time hadoop jar /usr/hdp/2.6.4.5-2/hadoop-mapreduce/hadoop-mapreduce-client-jobclient-2.7.3.2.6.4.5-2-tests.jar TestDFSIO -D mapred.output.compress=false -read -nrFiles 50 -fileSize 1000
```

After recording the output from these test, don't forget to remove the test files:

```
hadoop jar /usr/hdp/2.6.4.5-2/hadoop-mapreduce/hadoop-mapreduce-client-jobclient-2.7.3.2.6.4.5-2-tests.jar TestDFSIO -clean
```

#### Test 100GB

To test `write` for 100GB:

```
time hadoop jar /usr/hdp/2.6.4.5-2/hadoop-mapreduce/hadoop-mapreduce-client-jobclient-2.7.3.2.6.4.5-2-tests.jar TestDFSIO -D mapred.output.compress=false -write -nrFiles 100 -fileSize 1000
```

To test `read` for 50GB:

```
time hadoop jar /usr/hdp/2.6.4.5-2/hadoop-mapreduce/hadoop-mapreduce-client-jobclient-2.7.3.2.6.4.5-2-tests.jar TestDFSIO -D mapred.output.compress=false -read -nrFiles 100 -fileSize 1000
```

After recording the output from these test, don't forget to remove the test files:

```
hadoop jar /usr/hdp/2.6.4.5-2/hadoop-mapreduce/hadoop-mapreduce-client-jobclient-2.7.3.2.6.4.5-2-tests.jar TestDFSIO -clean
```
