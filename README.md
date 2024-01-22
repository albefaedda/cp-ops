# cp-ops

cp-ops is a demo to show how to setup the management of Confluent Platform resources in a cp-ansible (VMs) installation making use of Confluent for Kubernetes (CFK) Operator, the KafkaRestClass and ArgoCD GitOps tool. 

## Prerequisites

To install the Confluent Platform, we'll use a simple cp-ansible playbook which installs some of the Confluent Platform components in AWS VMs. 

We'll use [Minikube](https://minikube.sigs.k8s.io/docs/start/) for a local installation of CFK Operator and KafkaRestClass. 

## Installation

### Install Confluent Platform

To install the Confluent Platform I'll be using a simple cp-ansible configuration with 3 Zookeeper instances, 3 Kafka broker instances, 1 Connect cluster with Datagen Connector plugin on a single VM and 1 instance of Schema Registry with no security enabled.

I launched 3 VMs on AWS, and craeted the cp-ansible `hosts.yml` configuration adding the DNS name of these VMs. To proceed with the installation of CP components, these are the steps: 

Check connection to VMs:
```sh
ansible -i hosts.yml all -m ping -e "SSH_PRIVATE_KEY=~/path/to/your/aws-key.pem"
```

Run ansible playbook: 
```sh
ansible-playbook -e "SSH_PRIVATE_KEY=~/path/to/your/ssh-key.pem" -i hosts.yml confluent.platform.all
```

This installation does also include a Metadata Server (MDS) which can be used to create resources in the Kafka Cluster, like Topics, RoleBindings, Schemas, etc, and is accessible using one of the kafka brokers endpoints at port 8090.

### Install Confluent for Kubernetes

Let's create a new namespace named `confluent` and change our context to use it: 

```sh 
kubectl create namespace confluent

kubectl config set-context --current --namespace confluent
```

To install CFK Operator,you can either download the [CFK helm charts](https://docs.confluent.io/operator/current/co-deploy-cfk.html#deploy-co-using-the-download-bundle) which would allow us to customize the operator configuration, if needed, or can directly instsall the Operator from the helm repository. 

If you have downloaded the package, you can now unpack the bundle using the command `tar -xf confluent-for-kubernetes-2.7.3.tar.gz` and after customizing it, you can proceed with the deployment. 

From the `helm` subdirectory to where we downloaded the package, we can now install CFK: 

```sh
helm upgrade --install confluent-operator \
  ./confluent-for-kubernetes \
  --namespace confluent
```

To install CFK operator from the helm repo, we can run the command: 

```sh
helm upgrade --install confluent-operator \
  confluentinc/confluent-for-kubernetes \
  --namespace <namespace>
```

This command deployes Confluent Operator as well as Confluent's Custom Resources Definitions (CRDs) for the different CP components. 
We can verify the deployment is successful by running the command `kubectl get pods` which should return the name of our operator: 

```sh
$ kubectl get pods
NAME                                      READY   STATUS 
pod/confluent-operator-7768774786-4s6p5   1/1     Running
```

The next step is to deploy the KafkaRestClass which connects to our cp-ansible cluster through MDS endpoint; to do that we deploy the CR below by using the `kubectl apply -f kafka-rest-class.yml` command: 

```yml
apiVersion: platform.confluent.io/v1beta1
kind: KafkaRestClass
metadata:
  name: default
  namespace: confluent
spec:
  kafkaRest:
    endpoint: http://<kafka-broker-endpoint>:8090
```

### Install ArgoCD

For the automation in management of Kafka resources, we now install ArgoCD GitOps tool in our Kubernetes `confluent` namespace. 
We will use the full installation `argo-cd/extended-install.yaml`. I have updated the yaml file to make sure the components are deployed in the namespace `confluent`.

To proceed with the deployment, we will run the following command: 
```sh
kubectl apply -n confluent -f argo-cd/extended-install.yaml
```

This installation uses self-signed certificates, to be able to access argo-cd we'll just use the `--insecure` flag on all CLI commands. 

### Expose ArgoCD outside of Kubernetes cluster

To install the ArgoCD CLI on a mac, we'll use brew: `brew install argocd`

By default, the Argo CD API server is not exposed with an external IP. To access the API server, choose one of the following techniques to expose the Argo CD API server:

Change the argocd-service type to LoadBalancer:
```sh
kubectl patch svc argocd-server -n confluent -p '{"spec": {"type": "LoadBalancer"}}'
```

Use Port Forwarding to access the service: 
```sh
kubectl port-forward svc/argocd-server -n confluent 8080:443
```

The API server can then be accessed using `https://localhost:8080`

The initial password for the admin account is auto-generated and stored as clear text in the field password in a secret named argocd-initial-admin-secret in your Argo CD installation namespace. You can simply retrieve this password using the argocd CLI:

```sh
argocd admin initial-password -n confluent
```

Using the username adming and the password from the previous command, we can now login using the following command: 

```sh
argocd login <ARGOCD_SERVER>
```

## Resources Management

Let's now see how we can use ArgoCD to manage the resources of our cp-ansible cluster by making use of CFK CRs for the definition of resources. 

### Register a cluster to deploy apps to

To register a cluster to deploy apps to, let's find the available clusters: 
```sh
kubectl config get-contexts -o name
```

Once we have identified the cluster from the above command, let's supply it to argoCD: 

If your target Kubernetes cluster is the same cluster where ArgoCD is running (this is also my case) then we'll use `https://kubernetes.default.svc` as Kubernetes API server address. 

```sh
argocd cluster add CONTEXTNAME
```

We can now create an app that ArgoCD will sync into the cluster from a git repo, by providing it with the git address and a path where to find the resources to sync in the cluster. 

### Create an application to sync from a Git repo 

```sh
argocd app create cp-resources --repo https://github.com/albefaedda/cp-ops.git --path resources --dest-server https://kubernetes.default.svc --dest-namespace confluent
```

Once the cp-resources application is created, you can now view its status:

```sh
argocd app get cp-resources
```

### Sync the application

The application status is initially in OutOfSync state since the application has yet to be deployed, and no Kubernetes resources have been created. To sync (deploy) the application, run:

```sh
argocd app sync cp-resources
```

#### Topics Sync

The first type of resources we sync in the cluster are KafkaTopic(s). The KafkaTopic CR will use the KafkaRestClass reference to be able to "find" the cluster where it will be created. The KafkaRestClass, called `default` in this example, has a reference to the MDS endpoint in the cp-ansible cluster - example:

```sh
apiVersion: platform.confluent.io/v1beta1
kind: KafkaTopic
metadata:
  name: tenant-b-topic-2
  namespace: confluent
spec:
  replicas: 1
  partitionCount: 1
  kafkaRestClassRef:
    name: default
```

#### Connectors Sync

The other type of resource we sync in the cp-ansible cluster are Connector(s). The Connector CR uses the Connect's REST endpoint to reference the target cluster where the Connector will be created - example: 

```sh
apiVersion: platform.confluent.io/v1beta1
kind: Connector
metadata:
  name: pageviews-connector
  namespace: confluent
spec:
  class: "io.confluent.kafka.connect.datagen.DatagenConnector"
  taskMax: 4
  connectRest:
    endpoint: http://<connect-cluster-endpoint>:8083
  configs:
    kafka.topic: "pageviews"
    quickstart: "pageviews"
    key.converter: "org.apache.kafka.connect.storage.StringConverter"
    value.converter: "org.apache.kafka.connect.json.JsonConverter"
    value.converter.schemas.enable: "false"
    max.interval: "100"
    iterations: "100"
```

To delete a resoource, in this case the connector, this would be the command: 
```sh
argocd app delete-resource cp-resources --kind Connector --resource-name pageviews-connector
```

The Kubernetes cluster will show us the resources that have been created by running the `kubectl get all` command:

```sh
$ kubectl get all
NAME                                                REPLICAS   PARTITION   STATUS    CLUSTERID                AGE  
kafkatopic.platform.confluent.io/pageviews          3          1           CREATED   vErfDdFSRMqsMSo9s38E6g   27m  
kafkatopic.platform.confluent.io/tenant-b-topic-0   1          1           CREATED   vErfDdFSRMqsMSo9s38E6g   7d  
kafkatopic.platform.confluent.io/tenant-b-topic-1   1          1           CREATED   vErfDdFSRMqsMSo9s38E6g   75m  
kafkatopic.platform.confluent.io/tenant-b-topic-2   1          1           CREATED   vErfDdFSRMqsMSo9s38E6g   75m  

NAME                                                  STATUS    CONNECTORSTATUS   TASKS-READY   AGE  
connector.platform.confluent.io/pageviews-connector   CREATED   RUNNING           4/4           5m  

```