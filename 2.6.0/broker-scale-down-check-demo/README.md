# What's new in AMQ Streams 2.6.0

## Prerequisites

* OpenShift cluster

## Install AMQ Streams

1. Install AMQ Streams 2.6.0 using one of the available methods.
   You can also alternatively use the Strimzi 0.38.0 release, which corresponds to AMQ Streams 2.6.0.

## Preparation

2. Create a new project named `myproject` and set it as the default namespace/project.
   If you decide to use a different namespace/project, you might need to adjust the labs accordingly.

3. Deploy a new Apache Kafka cluster and wait until it is ready.
   You can use the [`kafka.yaml`](./kafka.yaml) file from this repository:
   ```
   kubectl apply -f https://raw.githubusercontent.com/ShubhamRwt/what-is-new-in-amq-streams/main/2.6.0/broker-scale-down-check-demo/kafka-cruise-control.yaml
   ```

## Broker scale down check demo

4. Create a new topic `my-topic`.
   You can use the [`topic.yaml`](./topic.yaml) file from this repository:
   ```
   kubectl apply -f https://raw.githubusercontent.com/ShubhamRwt/what-is-new-in-amq-streams/main/2.6.0/broker-scale-down-check-demo/topic.yaml
   ```

5. Try scaling down the last broker by changing the replicas count in the Kafka custom resource.

6. You can now check how topic partition replicas are allocated across the brokers using the command:
   ```
   kubectl run -n myproject client -itq --rm --restart="Never" --image="quay.io/strimzi/kafka:latest-kafka-3.6.0" -- \
   sh -c "/opt/kafka/bin/kafka-topics.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --describe --topic my-topic; exit 0";

   ```
### Scaling down without moving out all replicas from the broker to be removed

7. Let's try to scale down `.spec.kafka.replicas` (no. of brokers) from 4 to 3, while all brokers have partition replicas assigned.

   ```
   apiVersion: kafka.strimzi.io/v1beta2
   kind: Kafka
   metadata:
     name: my-cluster
   spec:
       kafka:
       version: 3.6.0
       replicas: 3            # Changes the replicas to 3
       listeners:
   # ....    
   ```

   Now you can apply the updated Kafka custom resource and check if the broker are scaled down or not.

   You check the status of the Kafka CR using this command:
   ```
   kubectl get kafka my-cluster -n myproject -o yaml
   ```

   Since we didn't move the replicas from the broker to be removed, the scale down will fail and, you will be able to see these errors in the status of the Kafka custom resource
   ```yaml
   status:
     clusterId: S4kUmhqvTQegHCGXPrHe_A
     conditions:
     - lastTransitionTime: "2023-12-04T07:03:49.161878138Z"
       message: Cannot scale down brokers [3] because brokers [3] are not empty
       reason: InvalidResourceException
       status: "True"
       type: NotReady
   ```
   So the status is basically telling you that broker 3 is not empty, which makes the whole reconciliation fail. You can get rid of this error by reverting `.spec.kafka.replicas` in the Kafka resource.

### Scaling down after moving out all replicas from the broker to be removed

8. Let's try to scale down the broker now after emptying partition replicas from the broker to be removed.

   We can make use of the `KafkaRebalance` resource in Strimzi with `remove-broker` node configuration for this job.
   Here is an example `KafkaRebalance` resource.
   ```
   apiVersion: kafka.strimzi.io/v1beta2
   kind: KafkaRebalance
   metadata:
     name: my-rebalance
     labels:
       strimzi.io/cluster: my-cluster
   # no goals specified, using the default goals from the Cruise Control configuration
   spec:
     mode: remove-brokers
     brokers: [3]
   ```
   You can create this `KafkaRebalance` custom resource and this will allow Cruise Control to handle the task of rebalancing and moving the partition replicas from the broker that is going to be removed.
   ```
   kubectl create -f https://raw.githubusercontent.com/ShubhamRwt/what-is-new-in-amq-streams/main/2.6.0/broker-scale-down-check-demo/kafka-rebalance-remove-brokers.yaml -n myproject 
   ```
   Once the proposal is ready, you can approve it and let the rebalacing to be complete
   ```
   kubectl annotate kafkarebalance my-rebalance strimzi.io/rebalance=approve -n myproject
   ```

   For more in-depth information you can refer to our [Rebalancing cluster using Cruise Control](https://strimzi.io/docs/operators/latest/deploying#proc-generating-optimization-proposals-str) documentation.

   Once the rebalacing is done, you can validate/check if the topics are move from broker or not.
   ```shell
   Topic: my-topic	TopicId: bbX7RyTSSXmheaxSPyRIVw	PartitionCount: 3	ReplicationFactor: 3	Configs: min.insync.replicas=2,segment.bytes=1073741824,retention.ms=7200000,message.format.version=3.0-IV1
   	Topic: my-topic	Partition: 0	Leader: 2	Replicas: 2,0,1	Isr: 2,1,0
   	Topic: my-topic	Partition: 1	Leader: 2	Replicas: 2,1,0	Isr: 1,0,2
  	Topic: my-topic	Partition: 2	Leader: 1	Replicas: 1,0,2	Isr: 1,0,2
   ```

   Now you can scale down the broker and, it will happen flawlessly since the broker is empty.
