# cloud-hadoop-benchmarks

Use common Hadoop benchmarks to evaluate the performance of various cloud provider platforms.

## Overview

Cloudbreak simplifies the deployment of Hortonworks platforms in cloud environments such as Amazon Web Services, Microsoft Azure and Google Cloud Platform. Cloudbreak enables the enterprise to quickly run big data workloads in the cloud while optimizing the use of cloud resources.

Cloudbreak is also a great tool for automating the deployment of multiple clusters across different cloud platform providers.  This ensures our ability to deploy known HDP configurations while minimizing the vartions to just instance and/or storage types.

You can read more about Cloudbreak [here](https://hortonworks.com/open-source/cloudbreak/) and [here](https://docs.hortonworks.com/HDPDocuments/Cloudbreak/Cloudbreak-2.4.2/content/index.html).

There are a number of common benchmarking tools avaialble for Hadoop.  I will use TestDFISIO, TeraGen, TeraSort and TPC-DS.

## Prerequisites

### Cloudbreak

I recommend using Cloudbreak to deploy clusters using the provided blueprints and cluster definitions to achieve similar results to my testing.  However it's not required.

- Cloudbreak 2.8.0 TP
- Ambari 2.6.2.2
- HDP 2.6.5.0

You can create a local instance of Cloudbreak using Vagrant and Virtualbox.  I have a tutorial that covers how to do that: [Using Vagrant and Virtualbox to create a local instance of Cloudbreak 2.4.1](https://community.hortonworks.com/articles/194076/using-vagrant-and-virtualbox-to-create-a-local-ins.html).

If you follow my tutorial above, you can place all of the json files in this repo inside of the vagrant directory for your Cloudbreak deployer.  They will be accessible in the `/vagrant` directory making it easy to use them with the Cloudbreak CLI.

### Cloudbreak CLI

You should install the Cloudbreak CLI onto your Cloudbreak deployer node.  This will make automating the benchmarking much easier rather than using the UI Wizards.  The documentation walks you through that: [Installing Cloudbreak CLI](https://docs.hortonworks.com/HDPDocuments/Cloudbreak/Cloudbreak-2.4.2/content/cli-install/index.html).  Before using the CLI, you should regiser your Cloudbreak hostname, username and password with the CLI using the following command:

`cb configure --server <cloudbreak hostname> --username <cloudbreak username> --password <cloudbreak password>`

### Cloud Credentials

You need credentials for your cloud provider platform of choice.  You need to define these credentials within Cloudbreak.


## Running the benchmarks

You should log into the client node created by Cloudbreak using the ssh key you specified in the cluster definition.  All tests should be performed on the client node.

### Cluster Definitions

The cluster definitions included in this repo were generated with Cloudbreak using the Create Cluster wizard.  You can choose to recreate your own cluster definitions or you can use the attached cluster JSON files.  These files are used with the Cloudbreak CLI.

You should modify the cluster defintion json files to meet your needs.  Common sections that will need to be changed include:

- **tags**
  - **userDefinedTags** - You should update the tags appropriate for your usage
- **placement**
  - **availabilityZone** - You may want to use a differnet availability zone
  - **region** - You may want to use a different region
- **general**
  - **credentialName**- This should correspond to the credential name for your cloud provider
- **cluster**
  - **password** - You should change the Ambari admin password
- **instanceGroups**
  - **instanceType** - You may want to use different instance types
  - **volumetype** - You may want to use different storage types
  - **volumeCount** - You may want to use a different number of volumes
  - **volumeSize** - You may want to use a different size of volume
  - **securityGroupId** - This should correspond to an existing security group you have created for your cloud provider
- **network**
  - **subnetId** - This should correspond to an existing subnet you have created for your cloud provider
  - **vpcId** - This should correspond to an existing subnet you have created for your cloud provider

### Blueprints

I recommend using the provided blueprints for benchmark purposes.  However, you may want to perform A-B testing with small changes to the cluster layout or component configurations.  I have provided 2 hive-focused blueprints.  To register the blueprints, you need to be on your Cloudbreak deployer node and run the following:

```
cb blueprint create from-file --file hive-tpcds-blueprint.json --name hive-tpcds
cb blueprint create from-file --file hive-tpcds-blueprint-compute.json --name hive-tpcds
```

### Create Cluster

You can chose to use the Cloudbreak UI wizard to create clusters.  However, it's much faster to use the CLI.  To create a cluster using the CLI, run the following command:

`cb cluster create --cli-input-json <cluster definition json file> --name <cluster name>`

For example:

`cb cluster create --cli-input-json hive-tpcds-cluster-m4-xlarge.json --name myoung-hive-tpcds-2`

### Login to Client Node

Once the cluster is running, you should log into the client node.  The client node will be on the cluster details page in Cloudbreak.  Here is an exmaple:

![Image](https://github.com/Jaraxal/cloud-hadoop-benchmarks/blob/master/cb-cluster-details.png)

You should use the "cloudbreak" username and the key you specified in the cluster defintion.  For example:

`ssh -i ~/Downloads/cloudbreak-crashcourse.pem  cloudbreak@13.57.199.34`

You should run all of the benchmarks on the client node to minimize performance impact potential to other nodes in the cluster.  You should clone this repo onto the client node which will provide you the tpcds queries and benchmarks.

You may have to install `git`.

`yum install -y git`

`git clone https://github.com/Jaraxal/cloud-hadoop-benchmarks.git`

`cd cloud-hadoop-benchmarks`

Before running the benchmarks, you should create the required HDFS directories.

```
sudo -u hdfs hadoop fs -mkdir /user/cloudbreak
sudo -u hdfs hadoop fs -chown cloudbreak /user/cloudbreak
sudo -u hdfs hadoop fs -mkdir /benchmarks
sudo -u hdfs hadoop fs -chown cloudbreak /benchmarks

```

If you are using Cloudbreak to create yoru clusters for these benchmarks, consider using a Cloudbreak recipe to automate the steps above.  The Cloudbreak recipe looks like this:

```
#!/bin/bash

yum install -y git

cd /home/cloudbreak

git clone https://github.com/Jaraxal/cloud-hadoop-benchmarks.git

cd /home/cloudbreak/cloud-hadoop-benchmarks

sudo -u hdfs hadoop fs -mkdir /user/cloudbreak
sudo -u hdfs hadoop fs -chown cloudbreak /user/cloudbreak
sudo -u hdfs hadoop fs -mkdir /benchmarks
sudo -u hdfs hadoop fs -chown cloudbreak /benchmarks

chown -R cloudbreak:cloudbreak /home/cloudbreak

```

Once you have created the recipe, you should have the recipe run on the `client` node.  You should update the cluster template to do this.  Here is an excerpt of a template with the `recipeNames` section updated:

```
...
    {
      "parameters": {},
      "template": {
        "parameters": {},
        "instanceType": "m4.4xlarge",
        "volumeType": "gp2",
        "volumeCount": 1,
        "volumeSize": 100,
        "rootVolumeSize": 50,
        "awsParameters": {
          "encryption": {
            "type": "NONE"
          }
        }
      },
      "nodeCount": 1,
      "group": "client",
      "type": "CORE",
      "recoveryMode": "MANUAL",
      "recipeNames": [
        "download-benchmarks"
      ],
      "securityGroup": {
...
```

The section you need to change is:

```
      "recipeNames": [
        "download-benchmarks"
      ],
```

You may need to edit the name of the recipe based on what you named it when you created it in Cloudbreak.

### TestDFSIO

TestDFSIO has a `write` and a `read` component.  You should test both of these.

#### Test 50GB

To test `write` for 50GB:

```
time hadoop jar /usr/hdp/current/hadoop-mapreduce-client/hadoop-mapreduce-client-jobclient.jar TestDFSIO \
-D mapred.output.compress=false \
-write -nrFiles 50 -fileSize 1000

```

To test `read` for 50GB:

```
time hadoop jar /usr/hdp/current/hadoop-mapreduce-client/hadoop-mapreduce-client-jobclient.jar TestDFSIO \
-D mapred.output.compress=false \
-read -nrFiles 50 -fileSize 1000

```

After recording the output from these test, don't forget to remove the test files:

```
hadoop jar /usr/hdp/current/hadoop-mapreduce-client/hadoop-mapreduce-client-jobclient.jar TestDFSIO -clean

```

#### Test 100GB

To test `write` for 100GB:

```
time hadoop jar /usr/hdp/current/hadoop-mapreduce-client/hadoop-mapreduce-client-jobclient.jar TestDFSIO \
-D mapred.output.compress=false \
-write -nrFiles 100 -fileSize 1000

```

To test `read` for 100GB:

```
time hadoop jar /usr/hdp/current/hadoop-mapreduce-client/hadoop-mapreduce-client-jobclient.jar TestDFSIO \
-D mapred.output.compress=false \
-read -nrFiles 100 -fileSize 1000

```

After recording the output from these test, don't forget to remove the test files:

```
hadoop jar /usr/hdp/current/hadoop-mapreduce-client/hadoop-mapreduce-client-jobclient.jar TestDFSIO -clean

```

#### Test 500GB

To test `write` for 500GB:

```
time hadoop jar /usr/hdp/current/hadoop-mapreduce-client/hadoop-mapreduce-client-jobclient.jar TestDFSIO \
-D mapred.output.compress=false \
-write -nrFiles 500 -fileSize 1000

```

To test `read` for 500GB:

```
time hadoop jar /usr/hdp/current/hadoop-mapreduce-client/hadoop-mapreduce-client-jobclient.jar TestDFSIO \
-D mapred.output.compress=false \
-read -nrFiles 500 -fileSize 1000

```

After recording the output from these test, don't forget to remove the test files:

```
hadoop jar /usr/hdp/current/hadoop-mapreduce-client/hadoop-mapreduce-client-jobclient.jar TestDFSIO -clean

```

### TeraGen & TeraGen

The Terasort series of benchmarks is comprised of multiple tests: TeraGen, TeraSort, and TeraValidate.

***NOTE: The number of `mapred.map.tasks` and `mapred.reduce.tasks` should be 1 less than the total number CPUs across your worker and compute nodes.  For example, on clusters with 10 m4.xlarge worker nodes, that would be `39`.  On clusters with 10 m4.4xlarge worker nodes, that would be `159`.***

#### Test TeraGen 50GB


To test `TeraGen` for 50GB:

```
time hadoop jar /usr/hdp/current/hadoop-mapreduce-client/hadoop-mapreduce-examples.jar teragen \
-Dmapred.map.tasks=39 \
-Dmapred.reduce.tasks=39 \
-Dmapred.map.output.compress=true \
500000000 \
/user/cloudbreak/terasort-input-50

```

#### Test TeraSort 50GB

To test `TeraSort` for 50GB:

```
time hadoop jar /usr/hdp/current/hadoop-mapreduce-client/hadoop-mapreduce-examples.jar terasort \
-Dmapred.map.tasks=39 \
-Dmapred.reduce.tasks=39 \
-Dmapred.map.output.compress=true \
/user/cloudbreak/terasort-input-50 \
/user/cloudbreak/terasort-output-50

```

#### Test TeraValidate 50GB

To test `TeraValidate` for 50GB:

```
time hadoop jar /usr/hdp/current/hadoop-mapreduce-client/hadoop-mapreduce-examples.jar teravalidate \
-Dmapred.map.tasks=39 \
-Dmapred.reduce.tasks=39 \
/user/cloudbreak/terasort-output-50 \
/user/cloudbreak/terasort-report-50

```

#### Test TeraGen 100GB

To test `TeraGen` for 100GB:

```
time hadoop jar /usr/hdp/current/hadoop-mapreduce-client/hadoop-mapreduce-examples.jar teragen \
-Dmapred.map.tasks=39 \
-Dmapred.reduce.tasks=39 \
-Dmapred.map.output.compress=true \
1000000000 \
/user/cloudbreak/terasort-input-100

```

#### Test TeraSort 100GB

To test `TeraSort` for 100B:

```
time hadoop jar /usr/hdp/current/hadoop-mapreduce-client/hadoop-mapreduce-examples.jar terasort \
-Dmapred.map.tasks=39 \
-Dmapred.reduce.tasks=39 \
-Dmapred.map.output.compress=true \
/user/cloudbreak/terasort-input-100 \
/user/cloudbreak/terasort-output-100

```

#### Test TeraValidate 100GB

To test `TeraValidate` for 100GB:

```
time hadoop jar /usr/hdp/current/hadoop-mapreduce-client/hadoop-mapreduce-examples.jar teravalidate \
-Dmapred.map.tasks=39 \
-Dmapred.reduce.tasks=39 \
/user/cloudbreak/terasort-output-100 \
/user/cloudbreak/terasort-report-100

```

#### Test TeraGen 500GB

To test `TeraGen` for 500GB:

```
time hadoop jar /usr/hdp/current/hadoop-mapreduce-client/hadoop-mapreduce-examples.jar teragen \
-Dmapred.map.tasks=39 \
-Dmapred.reduce.tasks=39 \
-Dmapred.map.output.compress=true \
5000000000 \
/user/cloudbreak/terasort-input-500

```

#### Test TeraSort 500GB

To test `TeraSort` for 500GB:

```
time hadoop jar /usr/hdp/current/hadoop-mapreduce-client/hadoop-mapreduce-examples.jar terasort \
-Dmapred.map.tasks=39 \
-Dmapred.reduce.tasks=39 \
/user/cloudbreak/terasort-input-500 \
/user/cloudbreak/terasort-output-500

```

#### Test TeraValidate 500GB

To test `TeraValidate` for 500GB:

```
time hadoop jar /usr/hdp/current/hadoop-mapreduce-client/hadoop-mapreduce-examples.jar teravalidate \
-Dmapred.map.tasks=39 \
-Dmapred.reduce.tasks=39 \
/user/cloudbreak/terasort-output-500 \
/user/cloudbreak/terasort-report-500

```

#### Cleanup

If you are running multiple tests for different data sizes, you will need to cleanup the test data between each run.  After recording the output from these tests, don't forget to remove the test files if you plan additional benchmark runs or are on a long running cluster where you don't want to waste the space.

```
sudo -u hdfs hadoop fs -rm -r -skipTrash /user/cloudbreak/terasort-input-*
sudo -u hdfs hadoop fs -rm -r -skipTrash /user/cloudbreak/terasort-output-*
sudo -u hdfs hadoop fs -rm -r -skipTrash /user/cloudbreak/terasort-report-*

```

### TPC-DS

The TPC Benchmark DS (TPC-DS) is a decision support benchmark that models several generally applicable aspects of a decision support system, including queries and data maintenance.  You can read more about it here [TPC-DS](http://www.tpc.org/tpcds/)

I've chosen a subset of TPC-DS queries for my benchmarks.  I'm using `q4`, `q11`, `q29`, `q59`, `q74`, `q75`, `q78`, `q93`, `q97`.  These are generally longer running queries.  The TPC-DS benchmark suite provided here was forked from [Hortonworks hive-testbench](https://github.com/hortonworks/hive-testbench).

The `runSuite.pl` script runs each query 5 times.  You can modify the script to change the number of runs.

#### Test 50GB

```
./tpcds-build.sh
./tpcds-setup.sh 50
./runSuite.pl tpcds 50

```

#### Test 100GB

```
./tpcds-build.sh
./tpcds-setup.sh 100
./runSuite.pl tpcds 100

```

#### Test 500GB

```
./tpcds-build.sh
./tpcds-setup.sh 500
./runSuite.pl tpcds 500

```

#### Cleanup

Depending on the amount of storage you provided your cluster, you may need to cleanup the test data between runs of different data sizes if you are low on space.  If you are using a long running cluster where you don't want to waste the space, then you may need to cleanup the test data.

```
sudo -u hdfs hadoop fs -rm -r -skipTrash /tmp/all_*
sudo -u hdfs hadoop fs -rm -r -skipTrash /tmp/tpcds-generate

```

### Cluster Termination

Once you have completed a round of tests on a given configuration, terminate the clsuter via Cloudbreak.  Repeat the process for different instance types to compare relative performance.