# Elastic Opensearch - EKS - EFK

## create es domain

```bash
aws es create-elasticsearch-domain --cli-input-json  file://es_domain.json
```

## create the policy

```bash
aws iam create-policy --policy-name fluent-bit-policy --policy-document file://fluent-bit-policy.json
```

## create an IAM role for fluent-bit. We will deploy fluent-bit in a namespace as fluentbit

```bash
kubectl create namespace fluentbit

eksctl create iamserviceaccount \
    --name fluent-bit \
    --namespace fluentbit \
    --cluster eksdemo1  \
    --attach-policy-arn "arn:aws:iam::705413410358:policy/fluent-bit-policy" \
    --approve \
    --override-existing-serviceaccounts
```

## First, retrieve the Fluent Bit Role ARN

```bash
export FLUENTBIT_ROLE=$(eksctl get iamserviceaccount --cluster EKS_CLUSTER_NAME --namespace logging -o json | jq '.[].status.roleARN' -r)

eksctl get iamserviceaccount --cluster eksdemo1 --namespace fluentbit -o json | jq '.[].status.roleARN' -r

arn:aws:iam::705413410358:role/eksctl-eksdemo1-addon-iamserviceaccount-flue-Role1-9ZKK9HOZ8Y4O
```

## Next, get the Elasticsearch Endpoint

```bash
export ES_ENDPOINT=$(aws es describe-elasticsearch-domain --domain-name ES_DOMAIN_NAME --output text --query "DomainStatus.Endpoint")

aws es describe-elasticsearch-domain --domain-name es-domain --output text --query "DomainStatus.Endpoint"

search-es-domain-alb5aummhfbxeq2eeqpnimstiu.us-east-1.es.amazonaws.com
```

## Finally, update the Elasticsearch internal database. For this task, you can use postman

```bash
curl -sS -u "${ES_DOMAIN_USER}:${ES_DOMAIN_PASSWORD}" -X PATCH \
https://${ES_ENDPOINT}/_opendistro/_security/api/rolesmapping/all_access?pretty \
    -H 'Content-Type: application/json' \
    -d'
[
  {
    "op": "add", "path": "/backend_roles", "value": ["'${FLUENTBIT_ROLE}'"]
  }
]'

curl -sS -u "akhilmovva:Akhilmovva@286" -X PATCH \
https://search-es-domain-alb5aummhfbxeq2eeqpnimstiu.us-east-1.es.amazonaws.com/_opendistro/_security/api/rolesmapping/all_access?pretty \
    -H 'Content-Type: application/json' \
    -d'
[
  {
    "op": "add", "path": "/backend_roles", "value": ["'arn:aws:iam::705413410358:role/eksctl-eksdemo1-addon-iamserviceaccount-flue-Role1-9ZKK9HOZ8Y4O'"]
  }
]'
```

## Output

```text
{
  "status" : "OK",
  "message" : "'all_access' updated."
}
```

## fluentbit

Find CHANGE_ME and update with your configurations
Host : ES Domain host
AWS_Region : ES Domain region

```bash
kubectl apply -f fluentbit.yaml
```

## Access Kibana Dashboard

You can access Kibana Dashboard with the URL <https://ES_ENDPOINT/_plugin/kibana/>. Use master username and master password configured to login.

Once you log into Explore on your own select Connect to your Elasticsearch index on the welcome page. Then click on create Index pattern. Add *fluent-bit* as the index pattern and hit Next step.

## Links

- [faun](https://faun.pub/configure-aws-elasticsearch-service-with-eks-cluster-7cff1689e515)
