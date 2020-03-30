# AMQ Streams 1.4.0

## Install AMQ Streams 1.4.0

* Install AMQ Streams 1.4 from the Operator Hub to watch the whole cluster

## Create namespaces

* Create two namespaces `demo-europe` and `demo-na`
  * `oc new-project demo-na`
  * `oc new-project demo-europe`

## Deploy Kafka clusters

* We need to deploy the Kafka clusters in their respective namespaces
  * `oc apply -f 01-kafka-europe.yaml`
  * `oc apply -f 02-kafka-na.yaml`

## Deploy applications using the clusters

* Once the clusters are running, we can deploy some applications using them:
  * `oc apply -f 03-application-europe.yaml`
  * `oc apply -f 04-application-na.yaml`
* Notice that both applications are producing to the topic `my-topic`
* Check that the applications work and consume only the local messages

## Deploy the Mirror Makers

* Now we can deploy the Mirror Makers 2
  * `oc apply -f 05-mirror-maker-2-europe.yaml`
  * `oc apply -f 06-mirror-maker-2-na.yaml`
* MM2s should sync the topics
  * The Europe consumer should now be consuming from a copy of the NA topic and vice versa
  * Notice the topic names on both sides

## Read the data from Kafka Connect

* The mirrored data can be consumed also from Kafka Connect
* Deploy Kafka Connect
  * `oc apply -f 07-kafka-connect.yaml`
* Notice the annotation enabling the connector operator

## Deploy the Connector

* Check the list of connectors in the KafkaConnect status
* Deploy the connector
  * For test purposes we use only FileSink connector
  * `oc apply -f 08-file-sink-connector.yaml`
* Notice the status of the connector
* Check the file with the received messages
  * `oc exec $(oc get pods -o name | grep europe-connect) -ti -- tail -f /tmp/messages.sink.txt`