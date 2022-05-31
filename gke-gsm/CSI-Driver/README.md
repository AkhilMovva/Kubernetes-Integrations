# GKE Secrets

## csi driver

```bash
#deploy csi-driver files
kubectl apply -f secrets-store-csi-driver/rbac-secretproviderclass.yaml \
-f secrets-store-csi-driver/csidriver.yaml \
-f secrets-store-csi-driver/secrets-store.csi.x-k8s.io_secretproviderclasses.yaml \
-f secrets-store-csi-driver/secrets-store.csi.x-k8s.io_secretproviderclasspodstatuses.yaml \
-f secrets-store-csi-driver/secrets-store-csi-driver.yaml

# If using the driver to sync secrets-store content as Kubernetes Secrets, deploy the additional RBAC permissions
# required to enable this feature
kubectl apply -f secrets-store-csi-driver/rbac-secretprovidersyncing.yaml

# If using the secret rotation feature, deploy the additional RBAC permissions
# required to enable this feature
kubectl apply -f secrets-store-csi-driver/rbac-secretproviderrotation.yaml

kubectl get po --namespace=kube-system
kubectl get crd
```

## gcloud secrets provider

```bash
kubectl apply -f google-secrets-manager-provider/provider-gcp-plugin.yaml
```

## workload identity service account

The provider will use the workload identity of the pod that a secret is mounted onto when authenticating to the Google Secret Manager API. For this to work the workload identity of the pod must be configured and appropriate IAM bindings must be applied.

- Setup the workload identity service account.

```bash
$ export PROJECT_ID=gke-practice-341914

$ gcloud config set project $PROJECT_ID
gcloud config set project gke-practice-341914

# Create a service account for workload identity
$ gcloud iam service-accounts create gke-workload
gcloud iam service-accounts create gke-workload

# Allow "default/mypod" to act as the new service account
$ gcloud iam service-accounts add-iam-policy-binding \
    --role roles/iam.workloadIdentityUser \
    --member "serviceAccount:$PROJECT_ID.svc.id.goog[default/mypodserviceaccount]" \
    gke-workload@$PROJECT_ID.iam.gserviceaccount.com

gcloud iam service-accounts add-iam-policy-binding `
    --role roles/iam.workloadIdentityUser `
    --member "serviceAccount:gke-practice-341914.svc.id.goog[dev/nginx]" `
    gke-workload@gke-practice-341914.iam.gserviceaccount.com
```

- Create a secret that the workload identity service account can access

```bash
# Create a secret with 1 active version
$ echo "testing gcp secrets" > secret.data
$ gcloud secrets create testsecret --replication-policy=automatic --data-file=secret.data
$ rm secret.data

# grant the new service account permission to access the secret
$ gcloud secrets add-iam-policy-binding testsecret \
    --member=serviceAccount:gke-workload@$PROJECT_ID.iam.gserviceaccount.com \
    --role=roles/secretmanager.secretAccessor

gcloud secrets add-iam-policy-binding apisecret `
    --member=serviceAccount:gke-workload@gke-practice-341914.iam.gserviceaccount.com `
    --role=roles/secretmanager.secretAccessor

```

- Try it out the [example](https://github.com/GoogleCloudPlatform/secrets-store-csi-driver-provider-gcp/tree/main/examples) which attempts to mount the secret "test" in $PROJECT_ID to /var/secrets/good1.txt and /var/secrets/good2.txt

```bash
$ ./scripts/example.sh
$ kubectl exec -it mypod /bin/bash
root@mypod:/# ls /var/secrets
```

## Links

- Git - [secrets-store-csi-driver](https://github.com/kubernetes-sigs/secrets-store-csi-driver)
- Git - [secrets-store-csi-driver-provider-gcp](https://github.com/GoogleCloudPlatform/secrets-store-csi-driver-provider-gcp)
- Official - [secrets-store-csi-driver](https://secrets-store-csi-driver.sigs.k8s.io/introduction.html)
- Helpful - [Resource](https://github.com/GoogleCloudPlatform/secrets-store-csi-driver-provider-gcp/issues/37)
