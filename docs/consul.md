## Consul Setup
Create Gossip Encryption Key
```
GOSSIP_ENCRYPTION_KEY=$(consul keygen)
```
Create certificates
```
cfssl gencert -initca ca/ca-csr.json | cfssljson -bare ca
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca/ca-config.json \
  -profile=default \
  ca/consul-csr.json | cfssljson -bare consul
```
Create Secret with Key and Certificates
```
kubectl create secret generic consul \
  --from-literal="gossip-encryption-key=${GOSSIP_ENCRYPTION_KEY}" \
  --from-file=ca.pem \
  --from-file=consul.pem \
  --from-file=consul-key.pem
```
Create ConfigMap
```
kubectl create configmap consul --from-file=configs/consul-server.json
```
Create Services
```
kubectl apply -f services/consul.yaml
```
Create StatefulSet
```
kubectl apply -f statefulsets/consul.yaml
```
Watch until pods have are running
```
kubectl get pods -w
```
Check consul-2 log to make sure it has joined the cluster and determine the leader
```
kubectl logs consul-2 -f
```
Connect workstation to the consul leader
If the leader is consul-0
```
kubectl port-forward consul-0 8500:8500
```
Create master token - Do This Once
```
curl --request PUT http://127.0.0.1:8500/v1/acl/bootstrap
```
 - The output is the master token which will be used to create the agent token in the next step</br>
 - Consul servers need to be restarted to start using the new token</br>
 - Do this one at a time until the 3 servers have been restarted</br>
 - Use kubectl logs consul-x -f to make sure a new leader is found after each restart</br>
 - If pods are restarted too quickly, a quorum will be lost</br>
```
kubectl delete pod consul-0
```

After restarts are completed, connect workstation to any consul server
```
kubectl port-forward consul-0 8500:8500
```
Create agent token and ACL - Do This Once
```
curl \
  --request PUT \
  --header "X-Consul-Token: $ACL_MASTER_TOKEN" \
  --data \
'{
  "Name": "Agent Token",
  "Type": "client",
  "Rules": "node \"\" { policy = \"write\" } service \"\" { policy = \"read\" }"
}' http://127.0.0.1:8500/v1/acl/create
```
 - The output of the command is the acl_agent_token

Add token to each consul server - Do not create new tokens
```
curl \
  --request PUT \
  --header "X-Consul-Token: $ACL_MASTER_TOKEN" \
  --data \
'{
  "Token": "$ACL_AGENT_TOKEN"
}' http://127.0.0.1:8500/v1/agent/token/acl_agent_token
```
Check each server's logs to make sure they are connected to the cluster
```
kubectl logs consul-0
```
The following should be in the logs
```
[INFO] Updated agent's ACL token "acl_agent_token"
[INFO] agent: Synced node info
```

Create ACL for vault
```
curl --request PUT \
     --header "X-Consul-Token: ACL_MASTER_TOKEN" \
     --data \
'{
  "Name": "vault-token",
  "Type": "client",
  "Rules": "key \"vault/\" { policy = \"write\" } service \"vault\" { policy = \"write\" } node \"\" { policy = \"write\" } agent \"\" { policy = \"write\" } session \"\" { policy = \"write\" }"
}' http://127.0.0.1:8500/v1/acl/create
```
Output is the VAULT_CONSUL_TOKEN for Vault
```
{"ID":"0e2cf500-e793-65e6-58fd-68fb0e578dc1"}
```

[Setup and Configure Vault](https://github.com/jsmickey/kubernetes-consul-vault/blob/master/docs/vault.md)
