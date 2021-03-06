# How to use for AWS integration  (Use storageclass)
1 Create for zookeeper
```
  kubectl create -f ./zookeeper/
``` 
2 Check

   2.1 Check with ./zookeeper/test.sh
  
   2.2 Check the AWS EC2 - Volume
  
       In AWS, 10 3G images are created
      
3 Create kafka
```
   kubectl create -f .
```
4 Check

   4.1 Create successful (already exists error is fine)
  
   4.2 kubectl get pods,svc,statefulset -n kafka
  
   4.3 3 additional 3G images are created in AWS
  
5 Test with test folder

6 Restart computer, the message still exists

#

# Below is old usage:

# Kafka as Kubernetes StatefulSet

Example of three Kafka brokers depending on five Zookeeper instances.

To get consistent service DNS names `kafka-N.broker.kafka`(`.svc.cluster.local`), run everything in a [namespace](http://kubernetes.io/docs/admin/namespaces/walkthrough/):
```
kubectl create -f 00namespace.yml
```

## Set up volume claims

You may add [storage class](http://kubernetes.io/docs/user-guide/persistent-volumes/#storageclasses)
to the kafka StatefulSet declaration to enable automatic volume provisioning.

Alternatively create [PV](http://kubernetes.io/docs/user-guide/persistent-volumes/#persistent-volumes)s and [PVC](http://kubernetes.io/docs/user-guide/persistent-volumes/#persistentvolumeclaims)s manually. For example in Minikube.

```
./bootstrap/pv.sh
kubectl create -f ./bootstrap/pvc.yml
# check that claims are bound
kubectl get pvc
```

## Set up Zookeeper

There is a Zookeeper+StatefulSet [blog post](http://blog.kubernetes.io/2016/12/statefulset-run-scale-stateful-applications-in-kubernetes.html) and [example](https://github.com/kubernetes/contrib/tree/master/statefulsets/zookeeper),
but it appears tuned for workloads heavier than Kafka topic metadata.

The Kafka book (Definitive Guide, O'Reilly 2016) recommends that Kafka has its own Zookeeper cluster,
so we use the [official docker image](https://hub.docker.com/_/zookeeper/)
but with a [startup script change to guess node id from hostname](https://github.com/solsson/zookeeper-docker/commit/df9474f858ad548be8a365cb000a4dd2d2e3a217).

Zookeeper runs as a [Deployment](http://kubernetes.io/docs/user-guide/deployments/) without persistent storage:
```
kubectl create -f ./zookeeper/
```

If you lose your zookeeper cluster, kafka will be unaware that persisted topics exist.
The data is still there, but you need to re-create topics.

## Start Kafka

Assuming you have your PVCs `Bound`, or enabled automatic provisioning (see above), go ahead and:

```
kubectl create -f ./
```

You might want to verify in logs that Kafka found its own DNS name(s) correctly. Look for records like:
```
kubectl logs kafka-0 | grep "Registered broker"
# INFO Registered broker 0 at path /brokers/ids/0 with addresses: PLAINTEXT -> EndPoint(kafka-0.broker.kafka.svc.cluster.local,9092,PLAINTEXT)
```

## Testing manually

There's a Kafka pod that doesn't start the server, so you can invoke the various shell scripts.
```
kubectl create -f test/99testclient.yml
```

See `./test/test.sh` for some sample commands.

## Automated test, while going chaosmonkey on the cluster

This is WIP, but topic creation has been automated. Note that as a [Job](http://kubernetes.io/docs/user-guide/jobs/), it will restart if the command fails, including if the topic exists :(
```
kubectl create -f test/11topic-create-test1.yml
```

Pods that keep consuming messages (but they won't exit on cluster failures)
```
kubectl create -f test/21consumer-test1.yml
```

## Teardown & cleanup

Testing and retesting... delete the namespace. PVs are outside namespaces so delete them too.
```
kubectl delete namespace kafka
rm -R ./data/ && kubectl delete pv datadir-kafka-0 datadir-kafka-1 datadir-kafka-2
```
