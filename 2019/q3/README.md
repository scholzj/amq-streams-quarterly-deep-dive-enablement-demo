# AMQ Streams 1.2.0

**Since 1.2.0, only OpenShift Container Platform 3.11 or newer is supported.**

* Checkout this repository which will be used during the lab:
  * `git clone https://github.com/scholzj/amq-streams-quarterly-deep-dive-enablement-demo.git`
* Go to the `q3` directory
  * `cd q3`
* Start you OpenShift cluster
  * **You should use OpenShift Container Platform 3.11 or higher**
  * Run `minishift start` or `oc cluster up`
* Login as cluster administrator
  * `oc login -u system:admin`
* If you are not in a project `myproject`, create it and set it as active
  * `oc new-project myproject`
  * `oc project myproject`

## Install AMQ Streams 1.2.0

_(You will have to configure pull secrets for the Red Hat registry if you use OKD instead of OCP)_

* `oc apply -f 01-install/`

## Deploy Kafka cluster

* `oc apply -f 02-kafka.yaml`

## Status sub-resource

* Once the Kafka cluster is deployed, you can check the status
  * Do `oc get kafka my-cluster -o yaml` and notice the last section of the output
  * Check the conditions
  * Check the addresses
* Modify the Kafka cluster to use external listener using OpenShift routes (e.g. using `oc edit kafka my-cluster`)
```yaml
  kafka:
    # ...
    listeners:
      # ...
      external:
        type: route
```
*  Notice how the status has been upgraded

_Optionally_

* Set the resources to number larger than what is the amount of CPU / resources you have (e.g. using `oc edit kafka my-cluster`)
```yaml
  kafka:
    # ...
    resources:
      requests:
        memory: 200Gi
        cpu: 20000m
```
* Wait for the reconciliation to fail and check that `conditions` field in the `status` sub-resource

## Handling of invalid fields

* Add following section to the Kafka custom resource (e.g. using `oc edit kafka my-cluster`)
```yaml
spec:
  kafka:
    # ...
  zookeeper:
    # ...
  entityOperator:
    # ...
  invalid:
    someKey: someValue
```
* Check the CO logs to see the warnings about the unknown fields
  * `oc logs $(oc get pod -l name=strimzi-cluster-operator -o=jsonpath='{.items[0].metadata.name}')`
* When finished, delete the cluster
  * `oc delete kafka my-cluster`

## Adding and removing JBOD volumes

* Deploy a Kafka cluster which will be using JBOD storage
  * `oc apply -f kafka-jbod.yaml`
* Wait for the cluster to be deployed
* Add more volumes to the brokers
  * Or using `oc edit` by adding following section to the `kafka.storage` section
```yaml
      - id: 1
        type: persistent-claim
        size: 100Gi
        deleteClaim: true
```
* Wait for the update to finish

_Optionally_

* Try to remove the volume again and see how it is removed

## Volume Resizing

_This step has to be executed only in environments which support storage resizing (i.e. not on Minishift or Minikube). You can try it for example on OCP4 in AWS._

* Make sure your cluster supports volume resizing
* Use `oc edit kafka my-cluster` and change the size of one of the volumes (e.g. from `100Gi` to `110Gi`)
* Watch as the PVs / PVCs get resized and wait for the rolling update to resize the filesystem

## HTTP Bridge

* Deploy the HTTP Bridge and wait for it to be ready
  * `oc apply -f 03-http-bridge.yaml`
* Expose the bridge using OpenShift Route to allow it being used from our local computers
  * `oc apply -f 04-http-bridge-route.yaml`
  * Get the address of the route `export BRIDGE="$(oc get routes my-bridge -o jsonpath='{.status.ingress[0].host}')"`

### Sending messages

* Send messages using a simple `POST` call
  * Notice the `content-type` header which is important!
```sh
curl -X POST $BRIDGE/topics/my-topic \
  -H 'content-type: application/vnd.kafka.json.v2+json' \
  -d '{"records":[{"key":"message-key","value":"message-value"}]}' \
  | jq
```

### Receiving messages

* First, we need to create a consumer
```sh
curl -X POST $BRIDGE/consumers/my-group \
  -H 'content-type: application/vnd.kafka.v2+json' \
  -d '{
    "name": "consumer1",
    "format": "json",
    "auto.offset.reset": "earliest",
    "enable.auto.commit": "false",
    "fetch.min.bytes": "512",
    "consumer.request.timeout.ms": "30000"
  }' \
  | jq
```

* Then we need to subscribe the consumer to the topics it should receive from
```sh
curl -X POST $BRIDGE/consumers/my-group/instances/consumer1/subscription \
  -H 'content-type: application/vnd.kafka.v2+json' \
  -d '{"topics": ["my-topic"]}'
```

* And then we can consume the messages (can be called repeatedly - e.g. in a loop)
```sh
curl -X GET $BRIDGE/consumers/my-group/instances/consumer1/records \
  -H 'accept: application/vnd.kafka.json.v2+json' \
  | jq
```

### Combining with Kafka clients

* You can of course also combine the HTTP Bridge clients with the Kafka clients
* Consume the messages set through the HTTP Bridge using the `kafka-console-consumer`:
```sh
oc run kafka-consumer -ti --image=registry.redhat.io/amq7/amq-streams-kafka-22:1.2.0 --rm=true --restart=Never -- bin/kafka-console-consumer.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --topic my-topic --from-beginning
``` 

* Or produce messages using the `kafka-console-producer` and consume them with the HTTP Bridge:
  * Remember to send valid JSON messages
```sh
oc run kafka-producer -ti --image=registry.redhat.io/amq7/amq-streams-kafka-22:1.2.0 --rm=true --restart=Never -- bin/kafka-console-producer.sh --broker-list my-cluster-kafka-bootstrap:9092 --topic my-topic
```

### Committing the messages

* At the end we should close the consumer
```sh
curl -X POST $BRIDGE/consumers/my-group/instances/consumer1/offsets
```

### Closing the consumer

* At the end we should close the consumer
```sh
curl -X DELETE $BRIDGE/consumers/my-group/instances/consumer1
```