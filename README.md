[![MPL-2.0 License](https://img.shields.io/badge/license-MPL_2.0-blue.svg)](https://www.mozilla.org/en-US/MPL/2.0/)

# Networks Diff-study deployment

Clone the repositories:

* network-diff-deployment  (this repository)
* network-diff-server
* network-diff-study-server
* network-diff-study-app

## Cassandra install

Download the last version of [Cassandra](http://www.apache.org/dyn/closer.lua/cassandra/3.11.5/apache-cassandra-3.11.5-bin.tar.gz)

In order to be accessible to the clients, Cassandra has to be bind to one the ip address of the machine.  The following variables have to be modified in conf/cassandra.yaml before starting the Cassandra daemon.

```yaml
seed_provider:
    - class_name: org.apache.cassandra.locator.SimpleSeedProvider
      parameters:
          - seeds: "<YOUR_IP>"

listen_address: "<YOUR_IP>"

rpc_address: "0.0.0.0"

broadcast_rpc_address: "<YOUR_IP>"
```

### Cassandra scheme setup

```bash
cqlsh <YOUR_IP>
```

To create keyspaces in a single node cluster:

```sql
CREATE KEYSPACE IF NOT EXISTS iidm WITH REPLICATION = { 'class' : 'SimpleStrategy', 'replication_factor' : 1 };
CREATE KEYSPACE IF NOT EXISTS geo_data WITH REPLICATION = { 'class' : 'SimpleStrategy', 'replication_factor' : 1};
CREATE KEYSPACE IF NOT EXISTS diffstudy WITH REPLICATION = { 'class' : 'SimpleStrategy', 'replication_factor' : 1 };
```

Then download the following files:
```html
https://github.com/powsybl/powsybl-network-store/blob/master/network-store-server/src/main/resources/iidm.cql
https://github.com/powsybl/powsybl-geo-data/blob/master/geo-data-server/src/main/resources/geo_data.cql
https://github.com/itesla/network-diff-study-server/blob/main/src/main/resources/diffstudy.cql
```

Finally, type in the cqlsh shell these commands:
```bash 
source 'iidm.cql';
source 'geo_data.cql';
source 'diffstudy.cql';
```

## Docker compose deployment

To build the Docker images for network-diff-server, open a terminal in the network-diff-server directory and execute
```bash 
mvn clean install jib:dockerBuild -Djib.to.image=network-diff-server

```
Note1: network-diff-server needs the iidm-diff module, which is currently in branch networks_diff of powsybl-core. 
To be able to compile network-diff-server, switch to that branch, change the module version to 3.7.1 in pom.xml, and 
```bash 
cd powsybl-core/iidm/iidm-diff
mvn clean install
```

Note2: network-diff-server wants the network-store-client module, which is part of powsybl-network-store and is not available yet from the maven repository;
To compile powsybl-network-store, clone the [powsybl-network-store](https://github.com/powsybl/powsybl-network-store) repository and then
 ```bash 
cd powsybl-network-store
mvn clean install
```

To build the Docker images for network-diff-study-server, open a terminal in the network-diff-study-server directory and execute
```bash 
mvn clean install jib:dockerBuild -Djib.to.image=network-diff-study-server

```
Edit the cassandra.properties file and make the "cassandra.contact-points" property point to your Cassandra server

Note : When using docker-compose for deployment, your host machine is accessible from the containers thought the ip address
`172.17.0.1`, so to make Cassandra accessible from the deployed containers, '<YOUR_IP>' in the "Cassandra section" above should be set to `172.17.0.1`. 
Unfortunately, with a centos+docker (so: a local installed Cassandra that listens from ip 172.17.0.1 and a network-store-service in a docker configured to point to cassandra using 172.17.0.1 ) 
but it does not seem to work: "Error message is: "All host(s) tried for query failed (tried: /172.17.0.1:9042)"
To setup a demo for the project, Cassandra was installed on a separate server, configured to listen from its ip (ref. related instructions, above), 
and cassandra.properties edited, accordingly.

Install the orchestration tool docker-compose then execute :

```bash 
cd docker-compose
docker-compose up
```

You can now access to the swagger UI of all the available services:

Swagger UI:
```html
http://localhost:8080/swagger-ui.html  // network-store-server
http://localhost:5000/swagger-ui.html  // case-server
http://localhost:5003/swagger-ui.html  // network conversion server
http://localhost:6007/swagger-ui.html  // network-diff-server
http://localhost:6008/swagger-ui.html  // network-diff-study-server
```
Kibana management UI:
```html
http://localhost:5601
```
In order to show documents in the case-server index with Kibana, you must first create the index pattern ('Management' page) : case-server*


## Network-diff-study-app

Network-diff-study-app is an Angular based web application; to execute it, you need first to install Angular CLI (https://angular.io/cli),
then

```bash 
cd network-diff-study-app
npm update
ng serve
```
In your browser, open http://localhost:4200/ to see the application run.

Currently, the network-diff-study-app allows you 

1) to list and delete the studies in the db (Diffstudy: list)
2) to create a new diff-study, with a name for the case, a description and the two cases that you are going to compare (Diffstudy: create) 
3) to compare a voltage level between the two network in a study and display the differences (Diffstudy: compare).   

At the current stage, you cannot upload network data via the application, please use the "import a case in the public directory" service from the case-server service http://localhost:5000/swagger-ui.html swagger UI, instead.

Once networks are in the case-server, you can search for them in the "Diffstudy: create" form: try typing * in the Case1Uuid/Case2Uuid fields (search service backed by elasticsearch).

Note: web application tested with Google Chrome
