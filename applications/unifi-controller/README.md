## Generate TLS Certificates
To generate the required certificates use cfssl (or you're tool of choice).

First generate the root certificate:
```
cfssl gencert -initca certs/config/ca-csr.json | cfssljson -bare certs/ca
```
Now generate the mongodb certificate:
```
cfssl gencert \
     -ca=certs/ca.pem \
     -ca-key=certs/ca-key.pem \
     -config=certs/config/ca-config.json \
     -profile=default \
     certs/config/mongodb-csr.json | cfssljson -bare certs/mongodb
```
Merge certificate and private key:
```
cat mongodb.pem mongodb-key.pem > mongodb-full.pem
```
Now generate the mongodb-auth certificate:
```
cfssl gencert \
     -ca=certs/ca.pem \
     -ca-key=certs/ca-key.pem \
     -config=certs/config/ca-config.json \
     -profile=default \
     certs/config/mongodbauth-csr.json | cfssljson -bare certs/mongodbauth
```
Merge certificate and private key:
```
cat mongodbauth.pem mongodbauth-key.pem > mongodbauth-full.pem
```
Create secret
```
kubectl create secret generic mongodb-tls \
  --from-file=certs/ca.pem \
  --from-file=certs/mongodb-full.pem \
  --from-file=certs/mongodb.pem \
  --from-file=certs/mongodb-key.pem \
  --from-file=certs/mongodbauth.pem \
  --from-file=certs/mongodbauth-full.pem \
  -n unifi-controller
```
deploy everything:
```
kubectl apply -f mongodb/ -n unifi-controller
```
Connect to mongodb-0 and connect to the database:
```
kubectl exec --stdin --tty mongodb-0 -n unifi-controller -- /bin/bash
```
```
mongo --tls localhost --tlsCertificateKeyFile /data/tls/mongo/mongodb-full.pem --tlsCAFile /data/tls/mongo/ca.pem
```
Start a replica set:
```
rs.initiate(
   {
      _id: "unifi-mongodb-rs",
      version: 1,
      members: [
         { _id: 0, host : "mongodb-0.mongodb.unifi-controller.svc.cluster.local:27017" },
         { _id: 1, host : "mongodb-1.mongodb.unifi-controller.svc.cluster.local:27017" },
         { _id: 2, host : "mongodb-2.mongodb.unifi-controller.svc.cluster.local:27017" }
      ]
   }
)
```
Create root user:
```
use admin
db.createUser(
  {
    user: "mongodb-root-admin",
    pwd: passwordPrompt(), // or cleartext password
    roles: [ { role: "userAdminAnyDatabase", db: "admin" }, "readWriteAnyDatabase" ]
  }
)
```
authenticate with the newly created root user:
```
db.auth("mongodb-root-admin", "<PASSWORD>")
```
create unifi database
```
use unifi-db
```
Note that the database will be created when the first document is entered, so insert some data:
```
db.new_collection.insert({init: "database" })
```
Create unifi database user
```
db.createUser(
  {
    user: "unifi-db-user",
    pwd: passwordPrompt(), // or cleartext password
    roles: [ { role: "readWrite", db: "unifi-db" }]
  }
)
```


https://switchit-conseil.com/2019/10/16/deploy-a-secured-high-availability-mongodb-replica-set-on-kubernetes/