# Delpoying Metron's auth demo on AWS.

This is a simple HOWTO to explain how to deploying Metron's auth demo on a single node running on AWS. To be successful with this project the following requirements must be met:

1. You have access to AWS and are able to deploy ami-58c7013a located in Asia Pacific(Sydney) region.
2. The minimum instance requirements are: 8 vCPUs, 32GB RAM, 400GB SSD
3. You have basic understanding how-to navigate in Ambari

All credits to Simon who prepared and uploaded the demos at  https://github.com/simonellistonball/metron-field-demos !!

## Getting Started

Login in Ambari. To reset the password follow https://community.hortonworks.com/questions/449/how-to-reset-ambari-admin-password.html

### Prerequisites

Before we start we should do the following:

In Ambari check if all services are running and start the ones that are not. Example HBase RegionServer will not start. This will require manual entry to start. If Metron Indexing service is in red do not worry about it (we will fix this further in the document).

Storm (changes)

* Storm Slots 0/4 free. add more supervisor.slots.ports [6700, 6701, 6702, 6703, 6704,  6705,  6706,  6707,  6709]
* Restart Storm

Elastic (changes)

* gateway_recover_after_data_nodes -  From 3 to 1
* index_number_of_replicas - From 2 to 0
* index_number_of_shards - From 4 to 1
* masters_also_are_datanodes - From false to true
* expected data nodes - 1
* Elastic stop/Elastic start

* Create - Storm View -- via Ambari
* Install - Kibana Load Template
* Install - Elastic Search Template 
* Install - Zeppelin Notebook Import

To Fix Metron Indexing please do the following steps as a root user:

```
 rpm -e --nodeps python-requests
 yum install python-pip
 pip install requests
 service ambari-agent restart

```
In Meton/Profiler change the settings "Period Duration to 1"

### Installing

Log in as root in the host. And download the demo code from:

```
mkdir /demo
cd /demo
git clone https://github.com/simonellistonball/metron-field-demos.git
```

```
[root@demo metron-field-demos]# vi setup-full-dev.sh
[root@demo metron-field-demos]# cat setup-full-dev.sh
#!/bin/sh

export METRON_HOST=demo.hortonworks.com
export ES_URL=http://demo.hortonworks.com:9200
export REST_URL=http://${METRON_HOST}:8082

```

```
[root@demo metron-field-demos]#. ./setup-full-dev.sh

```

Next step is to install Maven!!

```
cd~
wget "http://apache.mirror.digitalpacific.com.au/maven/maven-3/3.5.2/binaries/apache-maven-3.5.2-bin.tar.gz"
sudo mkdir /usr/local/maven
sudo tar xzf apache-maven-3.5.2-bin.tar.gz -C /usr/local/maven
cd /usr/local/maven
sudo ln -s apache-maven-3.5.2 latest
sudo ln -s latest default
cd /usr/local/bin
sudo ln -s /usr/local/maven/latest/bin/mvn mvn
```
mvn -version
Allow all users in the enfironment to use the latest:
```
sudo sh -c "echo export M2_HOME=/usr/local/maven/latest >> /etc/environment"
```

Next step would be to download and build Metron we need metron-data-management-0.4.2.jar

```
cd /demo
git clone https://github.com/apache/metron
cd metron
git checkout apache-metron-0.4.2-release
```

Build Metron with HDP 2.5 profile!
```
mvn clean package -DskipTests -T 2C -P HDP-2.5.0.0,mpack
```

Copy the metron-data-management-0.4.2.jar to the .m2 folder if it's not there

```
cp /demo/metron/metron-platform/metron-data-management/target/metron-data-management-0.4.2.jar /root/.m2/repository/org/apache/metron/metron-data-management/0.4.2/metron-data-management-0.4.2.jar
```

## Building the demo-loade

To build the demo-loader.jar you'll need to go to

```
cd /demo/metron-field-demos/auth/demo-loader
mvn clean package -T2C
cp target/demo-loader-*-uber.jar ../remote/
```

Next steps would be to prepare the rest of the scripts:

```
cd /demo/metron-field-demos/auth/remote
```

Update run.sh withthe following:
```
#!/bin/sh
source /etc/default/metron
./demo_loader.sh -e 1848158 -c config.json -z $ZOOKEEPER -hf hostfilter.txt
```

Replace the line starting with CP in the demo_loader.sh with:
```
CP=/demo/metron-field-demos/auth/remote/demo-loader-0.4.2-uber.jar
```

Edit the following line in bootstrap.sh 
```
$METRON_HOME/bin/zk_load_configs.sh -z $ZOOKEEPER -m PATCH -c PROFILER -pf profiler_patch.json
```

Next is to update the es.json file by adding the following updates. 

[root@demo auth]# diff es.json es.json.orig
```
171,173c171
<           "type": "text",
<           "index": true,
<           "fielddata": true
---
>           "type": "text"
176,178c174
<           "type": "text",
<           "index": true,
<           "fielddata": true
---
>           "type": "text"
```

### Loading data in Metron and ElasticSearch

Furst step would be to add the parser configuration. Please use the metron swagger UI to PUT the parser.json file.
the rest could be added via curl

```
echo "\nSetting up Enrichment"
curl -X POST -u metron:metron -d@enrichment.json -H 'Content-Type: application/json' $REST_URL/api/v1/sensor/enrichment/config/auth
echo "\nSetting up Indexing"
curl -X POST -u metron:metron -d@indexing.json -H 'Content-Type: application/json' $REST_URL/api/v1/sensor/indexing/config/auth
echo "\nConfigure ES template"
curl -X POST -d@es.json -H 'Content-Type: application/json' $ES_URL/_template/auth
echo "Setup kafka"
curl -X POST -u metron:metron -d@kafka.json -H 'Content-Type: application/json' $REST_URL/api/v1/kafka/topic
# do a run for Fun
#curl -u metron:metron ${REST_URL}/api/v1/storm/parser/start/auth
```

Run only onece (bootstrap.sh) and Kick off the data generation process

```
cd /demo/metron-field-demos/auth/remote
./bootstrap.sh
./run.sh
```

### The End

If all is good you could check the data in Metron (I'll upload the updated files as well)

