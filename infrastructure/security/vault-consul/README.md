# Generate Certificates
Create required certificates with something like cfssl:
```
cfssl gencert \
     -ca=certs/ca.pem \
     -ca-key=certs/ca-key.pem \
     -config=certs/config/ca-config.json \
     -profile=default \
     certs/config/vault-csr.json | cfssljson -bare certs/vault
```

Create secret:
```
kubectl create secret generic vault \
  --from-file=certs/ca.pem \
  --from-file=certs/vault.pem \
  --from-file=certs/vault-key.pem \
  -n vault-system
```


# Consul cluster setup

## Bootstrap ACL 
Bootstrap the consul cluster using the consul API:

```
kubectl exec -it consul-0 -n vault-system -- curl --request PUT http://127.0.0.1:8500/v1/acl/bootstrap
``` 
(Sample) response:
```
{"ID":"9623fbd5-7e54-b1e5-5b5d-e8159d4d2d4e","AccessorID":"189eb394-2e8b-d959-88f1-7a236c037ad3","SecretID":"<master token>","Description":"Bootstrap Token (Global Management)","Policies":[{"ID":"00000000-0000-0000-0000-000000000001","Name":"global-management"}],"Local":false,"CreateTime":"2020-07-02T19:26:04.606965891Z","Hash":"oyrov6+GFLjo/KZAfqgxF/X4J/3LX0435DOBy9V22I0=","CreateIndex":11,"ModifyIndex":11}
```

## Create agent token policy

Create the agent token policy using the consul API:

```
kubectl exec -it consul-0 -n vault-system -- curl \
    --request PUT \
    --header "X-Consul-Token: <master token>" \
    --data '{
        "Name": "agent-token",
        "Description": "Agent Token Policy",
        "Rules": "node_prefix \"\" { policy = \"write\"} service_prefix \"\" { policy = \"read\"} key_prefix \"lock/\" { policy = \"write\"}",
        "Datacenters": ["nldc01"]
        }' http://127.0.0.1:8500/v1/acl/policy
```
(Sample) Response:
```
{"ID":"<agent token>","Name":"agent-token","Description":"Agent Token Policy","Rules":"node_prefix \"\" { policy = \"write\"} service_prefix \"\" { policy = \"read\"} key_prefix \"lock/\" { policy = \"write\"}","Datacenters":["nldc01"],"Hash":"OJYCRgtiRwuz5Pst/qrzAgzQMgw0BXG5Svxs+r9r/xo=","CreateIndex":19,"ModifyIndex":19}
```

after that you can create the agent token with the newly created policy

```
kubectl exec -it consul-0 -n vault-system -- curl \
    --request PUT \
    --header "X-Consul-Token: <master token>" \
    --data '{
        "Description": "Agent token",
        "Policies": [{"ID": "9623fbd5-7e54-b1e5-5b5d-e8159d4d2d4e"}],
        "Local": true
        }' http://127.0.0.1:8500/v1/acl/token
```
(Sample) Response:
```
{"AccessorID":"f9804d7c-90d9-c959-f9ec-e753f1455533","SecretID":"<agent id>","Description":"Agent token","Policies":[{"ID":"9623fbd5-7e54-b1e5-5b5d-e8159d4d2d4e","Name":"agent-token"}],"Local":true,"CreateTime":"2020-07-02T19:44:51.301524271Z","Hash":"dHzO2llkPkoDiv4i4KkFCgM9LwKoP/daIM92f17olEQ=","CreateIndex":33,"ModifyIndex":33}
```

#### Apply ACL to agents

For all 3 consul nodes set the Agent Token via API.

```
kubectl exec -it consul-0 -n vault-system -- curl --request PUT --header "X-CONSUL-TOKEN: <master token>" --data '{"Token": "<agent token>"}' http://localhost:8500/v1/agent/token/acl_agent_token
```
### Update anonymous Token

In order to allow operation like `consul members` works without a token we can allow anonymous some operations.

First we create a policy that allows to read node infos

```
kubectl exec -it consul-0 -n vault-system -- curl \
    --request PUT \
    --header "X-Consul-Token: <master token>" \
    --data '{
        "Name": "list-all-nodes",
        "Description": "Anonymous node info policy",
        "Rules": "node_prefix \"\" { policy = \"read\" }"
        }' http://127.0.0.1:8500/v1/acl/policy
```
(Sample) Response:
```
{"ID":"efc03cd2-60b2-f16f-9ef3-69313480e76d","Name":"list-al-nodes","Description":"Anonymous node info policy","Rules":"node_prefix \"\" { policy = \"read\" }","Hash":"J7N2FZKEM9xiji5uVAEbAroBD/F3Eq8bgddLgVZCCHI=","CreateIndex":80,"ModifyIndex":80}
```

now add this policy to the anonymous token

```
kubectl exec -it consul-0 -n vault-system -- curl \
    --request PUT \
    --header "X-Consul-Token: <master token>" \
    --data '{
        "Description": "Anonymous Token - List Nodes",
        "Policies": [{"ID": "efc03cd2-60b2-f16f-9ef3-69313480e76d"}]
        }' http://127.0.0.1:8500/v1/acl/token/00000000-0000-0000-0000-000000000002
```
(Sample) Response:
```
{"AccessorID":"00000000-0000-0000-0000-000000000002","SecretID":"anonymous","Description":"Anonymous Token - List Nodes","Policies":[{"ID":"efc03cd2-60b2-f16f-9ef3-69313480e76d","Name":"list-al-nodes"}],"Local":false,"CreateTime":"2020-07-02T19:25:33.405344709Z","Hash":"s3E1TWLOMKz5YmMS+U5A3OO0U8OguG+ql+7T24i5p9s=","CreateIndex":5,"ModifyIndex":97}
```
## Create Vault ACL
```
kubectl exec -it consul-0 -n vault-system -- curl \
    --request PUT \
    --header "X-Consul-Token: d81a0dac-11ba-1e0c-aa05-8c5eda62558d" \
    --data '{
        "Name": "vault-key",
        "Description": "Vault Policy",
        "Rules": "node_prefix \"\" { policy = \"write\"} session_prefix \"\" { policy = \"write\"} key_prefix \"vault\" { policy = \"write\"} service \"vault\" { policy = \"write\" } agent_prefix \"\" { policy = \"write\" }",
        "Datacenters": ["nldc01"]
        }' http://127.0.0.1:8500/v1/acl/policy
```
(Sample) Response:
```
{"ID":"bbc0ca3a-eb8e-eade-48fa-42b4c519a7d3","Name":"vault-key","Description":"Vault Policy","Rules":"node_prefix \"\" { policy = \"write\"} session_prefix \"\" { policy = \"write\"} key_prefix \"vault\" { policy = \"write\"} service \"vault\" { policy = \"write\" } agent_prefix \"\" { policy = \"write\" }","Datacenters":["nldc01"],"Hash":"TAgbTtuDhIwf6SwQJ5RajuGhDyw/+JON28aZsp0K4/g=","CreateIndex":7230,"ModifyIndex":7230}
```
Now add a token to the newly create vault-key policy
```
kubectl exec -it consul-0 -n vault-system -- curl \
    --request PUT \
    --header "X-Consul-Token: <master token>" \
    --data '{
        "Description": "Vault Token",
        "Policies": [{"ID": "bbc0ca3a-eb8e-eade-48fa-42b4c519a7d3"}],
        "Local": true
        }' http://127.0.0.1:8500/v1/acl/token
```
(Sample) Response:
```
{"AccessorID":"8f2683db-29d8-ba5e-d31b-56c545868f73","SecretID":"<vault-token>","Description":"Vault Token","Policies":[{"ID":"bbc0ca3a-eb8e-eade-48fa-42b4c519a7d3","Name":"vault-key"}],"Local":true,"CreateTime":"2020-07-03T12:08:35.619553208Z","Hash":"2XL0XtLQwKgZnRSOmfy1Ka5oo//vfZ+NInwmyLLIJcY=","CreateIndex":7260,"ModifyIndex":7260}
```
Create a secret containing the vault token:
```
kubectl create secret generic vault-key --from-literal=vault-token=<vault-token> -n vault-system
```