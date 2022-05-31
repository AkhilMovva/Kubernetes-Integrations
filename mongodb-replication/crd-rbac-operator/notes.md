# MongoDB on Kubernetes using CRD, Operator and Rbac

## Install MongoDB K8s Operator

To install MongoDB, we're going to be using an open-source Kubernetes operator. Operator and MongoDB will be deployed in the same namespace. Let's start with it.

* Give it a name MongoDB and also important to add a label monitoring equal to Prometheus. Prometheus will only monitor namespaces that contain this label.
  
* Next, we need to create a custom resource definition for MongoDBCommunity. It extends Kubernetes and allows you to define a custom type that only be created by the corresponding operator. It will create a MongoDB cluster based on this definition and manage its lifecycle. Create crd.yaml file.

* Since it will require Kubernetes API server access, we need to create some RBAC policies. RBAC is a role-based access control system for Kubernetes. Create rbac folder and corresponding files.

* Then finally, the operator itself. It's going to be deployed as a simple deployment object. You can adjust a few parameters if you want, such as operator version, image repository, and others.

* Now let's move to the terminal and apply all of these files. I assume that you already have Kubernetes provisioned and kubectl configured to talk to the cluster.

```bash
kubectl apply -f crd/namespace.yaml
kubectl apply -f crd/crd.yaml
kubectl apply -f rbac
kubectl apply -f crd/operator.yaml
```

* Let's check if the operator is running. Also, you can search logs for any errors.

```bash
kubectl get pods -n mongodb
```

## Install MongoDB on K8s (Standalone/Single Replica)

There are a couple of ways to manage users in MongoDB. You can create users using the MongoDB operator or using the shell. I'll show you both ways along the way. Start with creating an admin user with the custom resource.

* First, we need to create a secret with a password. In this case, admin123.

* Now the main database configuration file. It's going to be a MongoDBComunity type. Then specify how many replicas do you want. If you use one member, the operator will create a standalone MongoDB instance. We will start with one and scale it up a little bit later. For now, only a single type is supported, which is ReplicaSet. Enterprise Operator supports various types, including sharded cluster. Community edition operator at this point only can deploy a cluster with multiple replicas only.

* We're done configuring the first deployment. Let's go to the terminal and apply it.

```bash
kubectl apply -f mongo/secret.yaml
kubectl apply -f mongo/pv.yml
kubectl apply -f mongo/mongodb.yaml
```

* It may take a few seconds to create your first MongoDB instance. Let's make sure that the pod is running and passing all the health checks.

```bash
kubectl get pods -n mongodb
```

* You can find persistent volume claims; we have one for 38 gigs and the second for two gigs used for logging by the operator.

```bash
kubectl get pvc -n mongodb
```

* Conveniently, the operator will generate a secret with the credentials and connection strings. You can get it from the secret. You have a standard connection string and a DNS Seed List Connection String.

```bash
kubectl get secret my-mongodb-admin-admin-user -o yaml -n mongodb
```

* You can grab the string and decode it using the base64 tool.

```bash
echo "HGC%#DG" | base64 -d
```

* Or, as a shortcut, you can use the jq command to parse the secret.

```bash
kubectl get secret my-mongodb-admin-admin-user -n mongodb -o json | jq -r '.data | with_entries(.value |= @base64d)'
```

* Now let's connect to the database. We are going to be using a mongosh shell. You can use the port forward command to be able to access MongoDB locally.

```bash
kubectl port-forward my-mongodb-0 27017 -n mongodb
```

* To connect, provide the username and a password and use localhost with the default port number.

```bash
mongosh "mongodb://admin-user:admin123@127.0.0.1:27017/admin?directConnection=true&serverSelectionTimeoutMS=2000"

mongosh " mongodb://<user-name>:<password>:<host_ip>"
```

* First, list all the available databases.

```bash
show dbs
```

* Now create a new user.

```bash
db.createUser(
  {
    user: 'aputra',
    pwd: 'devops123',
    roles: [ { role: 'readWrite', db: 'store' } ]
  }
);
```

* Then authenticate using its credentials.

```bash
db.auth('aputra', 'devops123')
```

* Create a new store database.

```bash
use store
```

* Try to insert a record using the insertOne command.

```bash
db.employees.insertOne({name: "Anton"})
```

* Then, retrieve all the records in the collection using the find function.

```bash
db.employees.find()
```

## Install MongoDB on K8s (Replica Set)

* To scale up, simply increase the number of members from 1 to 3.

`mongodb.yaml`

```yml
---
apiVersion: mongodbcommunity.mongodb.com/v1
kind: MongoDBCommunity
metadata:
  name: my-mongodb
  namespace: mongodb
spec:
  members: 3
...
```

* Then apply it. In a few minutes operator will spin up two more replicas and join them to the cluster.

```bash
kubectl apply -f k8s/mongodb/database/mongodb.yaml
```

* Now we have one primary instance and two replicas.

```bash
kubectl get pods -n mongodb
```

* Now we have one primary instance and two replicas. Since we still have the previous session, let's verify that replicas are up to date. You need to switch to the admin database first.

```bash
use admin
```

* Then, authenticate with admin credentials.

```bash
db.auth('admin-user', 'admin123')
```

* If you run the status, you find all the members.

```bash
rs.status()
```

* You can also check the replication if replicas are able to keep up with the primary.

```bash
rs.printSecondaryReplicationInfo()
```

* At this point, we have the MongoDB cluster ready for use. For the following example, we will secure MongoDB with TLS, but first, we need to clean up and delete the current deployment and persistent volume claims with corresponding volumes.

```bash
kubectl delete -f k8s/mongodb/internal/mongodb.yaml
kubectl delete pvc -l app=my-mongodb-svc
```

## Links

Referance - <https://antonputra.com/kubernetes/how-to-install-mongodb-on-kubernetes/>
