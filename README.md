## EFK (Elasticsearch-Fluentd-Kibana) stack on the Kubernetes cluster
---------------------------------------------------

In this repository, we will see how to collect application logs to EFK (Elasticsearch + Fluentd + Kibana) stack on a Kubernetes cluster.

1. **Elasticsearch**: Elasticsearch is a search engine based on the Lucene library. It provides a distributed, multitenant-capable full-text search engine with an HTTP web interface and schema-free JSON documents.

2. **Kibana**: Kibana is an open-source data visualization dashboard for Elasticsearch. It provides visualization capabilities on top of the content indexed on an Elasticsearch cluster. Users can create bar, line and scatter plots, or pie charts and maps on top of large volumes of data.

3. **Fluentd**: Fluentd is an open-source data collector, which lets you unify the data collection and consumption for better use and understanding of data. It is a logging agent that takes care of log collection, parsing, and distribution.

By combining these three tools EFK (Elasticsearch + Fluentd + Kibana) we get a scalable, flexible, easy to use log collection and analytics stack.

We will first create a namespace (efk-space) into which we install all of the EFK stack Kubernetes objects. Create the namespace using the command below:

```
kubectl create -f namespace/namespace-def.yaml
```

Then we will install a 3-node Elasticsearch cluster using Kubernetes StatefulSet. We define a headless service which is needed for the statefulset object. We define a couple of InitContainers in the StatefulSet definition file that runs before the main elasticsearch app container. The first InitContainer runs a chown command to change the owner and group of the Elasticsearch data directory to 1000:1000 (Elasticsearch user’s UID). The second InitContainer runs a command to increase the operating system’s limits on mmap counts, which by default may be too low, resulting in out of memory errors. We set a volumeClaimTemplates for the StatefulSet to create PersistentVolumes for the elasticsearh Pods. We define the storage class as **do-block-storage** since this Kubernetes cluster is created on the DigitalOcean Kubernetes cluster. You should change this value depending on where you are running the Kubernetes cluster. Use the following command to create the headless service and StatefulSet.

```
kubectl create -f elasticSearch/elasticsearch-headless-svc.yaml
kubectl create -f elasticSearch/elasticsearch-statefulset.yaml
```

The next step is to deploy the Kibana tool so that we can visualize the data stored in the Elasticsearch. To launch Kibana on Kubernetes, we will create a service, and a deployment consisting of one Pod replica. Here we use the NodePort service and the Kibana tool can be accessed on the port 31000 of the respective node on which the Kibana pod is scheduled. You can get the node IP address on which the Kibana pod is deployed using the command: `kubectl get pods -n efk-space -o wide`. Use the following command to create the Kibana service and deployment.

```
kubectl create -f kibana/kibana-service.yaml
kubectl create -f kibana/kibana-deployment.yaml
```

Use the command below to check the status of the deployment.

```
kubectl rollout status deployment/kibana --namespace=efk-space
```

Once the pod is running, access the Kibana interface using the following URL:

```
http://<node-IP-address>:31000
```

Replace `<node-IP-address>` with the IP address of the node on which the Kibana pod is running.


Now, we need to deploy Fluentd (Node level logging agent) on each Node in the Kubernetes cluster. We will set up Fluentd as a DaemonSet that runs a copy of a given Pod on each node in the Kubernetes cluster. In Kubernetes, containerized applications that log to 'stdout' and 'stderr' have their log streams captured and redirected to JSON files on the nodes. The Fluentd Pod will tail these log files, filter log events, transform the log data, and ship it off to the Elasticsearch logging that we deployed.
We need to create a ServiceAccount for the DaemonSet that the Fluentd Pods will use to access the Kubernetes API. We also define a ClusterRole to which we grant the get, list, and watch permissions on the pods and namespaces objects. Additionally, we define ClusterRoleBinding which binds the ClusterRole to the ServiceAccount. This grants the ServiceAccount the permissions listed in the ClusterRole. Use the following commands to create Fluentd DaemonSet, ServiceAccount, ClusterRole, and ClusterRoleBinding.

```
kubectl create -f fluentd/fluentd-serviceAccount.yaml
kubectl create -f fluentd/fluentd-clusterRole.yaml
kubectl create -f fluentd/fluentd-clusterRoleBinding.yaml
kubectl create -f fluentd/fluentd-daemonSet.yaml
```

Now, the EFK stack is deployed on the Kubernetes cluster. Now, access the Kibana interface and define index pattern to capture all the log data in the Elasticsearch cluster.
