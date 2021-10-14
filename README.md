[![MPL-2.0 License](https://img.shields.io/badge/license-MPL_2.0-blue.svg)](https://www.mozilla.org/en-US/MPL/2.0/)

# Networks diff-study

## Requirements and dependencies: backend services
Network diff-study depends on the services provided by these modules

* [powsybl-network-store](https://github.com/powsybl/powsybl-network-store.git)  (compile time, runtime)
* [powsybl-network-conversion-server](https://github.com/powsybl/powsybl-network-conversion-server.git) (runtime)
* [powsybl-case](https://github.com/powsybl/powsybl-case.git) (runtime)
* [geo-data](https://github.com/gridsuite/geo-data.git) (compile time, runtime)
* [report-server](https://github.com/gridsuite/report-server.git) (runtime)

Docker takes care of providing the correct docker images versions for the modules runtimes, so there is no need to build everything from te sources.
However, powsybl-network-store and geo-data modules must be cloned from github and built locally, before compiling the network diff-study related modules.

Please note that it is required a specific commit id, for the above modules (since they are not esplicitly versioned on github),

* powsybl-network-store:  3356b20b10446209e21748ecc697d26782e331c6
```bash 
git clone https://github.com/powsybl/powsybl-network-store.git
cd powsybl-network-store
git checkout 3356b20b10446209e21748ecc697d26782e331c6
mvn clean install -DskipTests
```
* geo-data: 4368cf809287e47d191a8070ba4bee798de850b2
```bash 
git clone https://github.com/gridsuite/geo-data.git
cd geo-data
git checkout 4368cf809287e47d191a8070ba4bee798de850b2
mvn clean install -DskipTests
```

## Requirements and dependencies: Cassandra installation and configuration

Download and install the 3.11.10 version of [Cassandra](https://www.apache.org/dyn/closer.lua/cassandra/3.11.10/apache-cassandra-3.11.10-bin.tar.gz)

In order to be accessible to the clients, Cassandra has to be bind to one the ip address of the machine.  

The following variables have to be modified in conf/cassandra.yaml before starting the Cassandra daemon.

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

Then, download the following files:
```html
https://github.com/powsybl/powsybl-network-store/blob/1e5ea17264483053e5ef31b8f53c048fe67f2a43/network-store-server/src/main/resources/iidm.cql
https://github.com/gridsuite/geo-data/blob/ab460f3553111df9f5738853ba39c3dda39af6c4/geo-data-server/src/main/resources/geo_data.cql
https://github.com/itesla/network-diff-study-server/blob/main/src/main/resources/diffstudy.cql
```

Finally, type in the cqlsh shell these commands:
```bash 
use iidm;
source 'iidm.cql';
use geo_data;
source 'geo_data.cql';
use diffstudy;
source 'diffstudy.cql';
```

## Requirements and dependencies: Postgres
Postgres is currently required by the report server. To install Postgres, use your system package manager or a dedicated docker container.

### Postgres schema setup

$ bin/psql postgres
$ create database report;

Then initialize the schemas for the databases:

$ \c report; then copy/paste [report.sql](https://github.com/gridsuite/report-server/blob/c3ec699bb1ccdebd4952a8a9f8b3f7c77ed9ac50/src/main/resources/report.sql) content to psql

## Networks diff-study: compiling

Clone the repositories from GitHub

* [network-diff-deployment](https://github.com/itesla/network-diff-deployment.git)  (this repository: docker-compose files and instructions)
* [network-diff-iidm](https://github.com/itesla/network-diff-iidm.git)  (java module)
* [network-diff-server](https://github.com/itesla/network-diff-server.git) (java module)
* [network-diff-study-server](https://github.com/itesla/network-diff-study-server.git) (java module)
* [network-diff-study-app](https://github.com/itesla/network-diff-study-app.git)  (angular module)

and build all the java modules, using Maven (order is important: network-diff-iidm before the others).
```bash 
cd MODULE-DIR
mvn clean install

```

## Networks diff-study: create the modules' docker images

To build the Docker images for network-diff-server, open a terminal in the network-diff-server directory and execute
```bash 
cd network-diff-server
mvn clean install jib:dockerBuild -Djib.container.creationTime=USE_CURRENT_TIMESTAMP -Djib.to.image=network-diff-server

```

To build the Docker images for network-diff-study-server, open a terminal in the network-diff-study-server directory and execute
```bash 
cd network-diff-study-server
mvn clean install jib:dockerBuild -Djib.container.creationTime=USE_CURRENT_TIMESTAMP -Djib.to.image=network-diff-study-server

```

## Networks diff-study: docker-compose configuration and deployment

Install, if not already available, the orchestration tool [docker-compose](https://docs.docker.com/compose/install/).

Edit the network-diff-deployment/docker-compose/cassandra.properties file and make the "cassandra.contact-points" property point to your Cassandra server

Note : When using docker-compose for deployment, your host machine is accessible from the containers thought the ip address
`172.17.0.1`, so to make Cassandra accessible from the deployed containers, '<YOUR_IP>' in the "Cassandra section" above should be set to `172.17.0.1`. 
Unfortunately, with a centos+docker (so: a local installed Cassandra that listens from ip 172.17.0.1 and a network-store-service in a docker configured to point to cassandra using 172.17.0.1 ) 
but it does not seem to work: "Error message is: "All host(s) tried for query failed (tried: /172.17.0.1:9042)"
To setup a demo for the project, Cassandra was installed on a separate server, configured to listen from its ip (ref. related instructions, above), 
and cassandra.properties edited, accordingly.

Set the "POSTGRESSERVER_IP", "POSTGRES_USER" and "POSTGRES_PWD" properties in network-diff-deployment/docker-compose/database.properties file, appropriately.

### Execute docker-compose

```bash 
cd network-diff-deployment/docker-compose
docker-compose up
```

When all the available services are up and running (this might take some time), their swagger UIs can be reached at these URLs:

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
Note: in order to show documents in the case-server index with Kibana, you must first create the index pattern ('Management' page) : case-server*


## Network-diff-study-app (dev environment)

Network-diff-study-app is an Angular based web application; to execute it, you need first to install Angular CLI (https://angular.io/cli),
then

```bash 
cd network-diff-study-app
npm install
npm start
```

After the frontend service has started, the Network-diff-study-app is available at http://localhost:4200

Currently, the network-diff-study-app allows to:

1) create a new diff-study, from two cases that you are going to compare.
2) list and delete the diff-studies available in the database.
3) define a zone for the diff-study as a list of substations.
4) compare a voltage level between the two network in a study and display the differences.   
5) compare a substation between the two network in a study and display the differences.
6) display the differences in a zone (list of substations), on a geo map.

At the current stage, you cannot upload network data directly using the app, please use the "import a case in the public directory" service from the case-server service http://localhost:5000/swagger-ui.html swagger UI, instead.

Once networks are in the case-server, you can search for them in the "Diffstudy: create" form: try typing * in the Case1Uuid/Case2Uuid fields (search service backed by elasticsearch).

Note: web application tested with Google Chrome, Firefox


# Backend services versions details

|   | GitHub commit | Docker image sha256 |
| ------------- | ------------- | ------------- |
| network-store server |  [1e5ea17264483053e5ef31b8f53c048fe67f2a43](https://github.com/powsybl/powsybl-network-store/commit/1e5ea17264483053e5ef31b8f53c048fe67f2a43)  | powsybl/network-store-server@sha256:a8991bbdfb6e1a0b23de9541b0a3d3852dee7be4f36e6c298b6175f4f292f54a |
| geo-data server |  [ab460f3553111df9f5738853ba39c3dda39af6c4](https://github.com/gridsuite/geo-data/commit/ab460f3553111df9f5738853ba39c3dda39af6c4)  | gridsuite/geo-data-server@sha256:53e7a6d1ef69dc4919305b9e185891040a9cbeef67d6a3149d36c4a3057132dd |
| network-conversion-server  |  [309262767f06ae7944d6dc11c47f943343cd9d23](https://github.com/powsybl/powsybl-network-conversion-server/commit/309262767f06ae7944d6dc11c47f943343cd9d23)  | powsybl/network-conversion-server@sha256:d9bddf6de10782075c660eda5b6d3599cff75621f8d47d848fe7863f16204037 |
| case-server  |  [1c0e6a0f0af86b434f74388a32c6cfe9f3d9ec6a](https://github.com/powsybl/powsybl-case/commit/1c0e6a0f0af86b434f74388a32c6cfe9f3d9ec6a)  | powsybl/case-server@sha256:0f938727ffabb38a956474717ec8948967a03f7088989e98986a554e9b7a3e79 |
| report-server  |  [c3ec699bb1ccdebd4952a8a9f8b3f7c77ed9ac50](https://github.com/gridsuite/report-server/commit/c3ec699bb1ccdebd4952a8a9f8b3f7c77ed9ac50)  | gridsuite/report-server@sha256:8ba8856ab749ed264916b2aea88c8e97035b941061b95fc836b42419d4c722fe |
