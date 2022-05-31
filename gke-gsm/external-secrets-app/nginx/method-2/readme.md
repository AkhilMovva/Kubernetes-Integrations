# GKE Secret Manager. Consuming secrets using external Secrets

## Installing external secrets

```bash
# $ helm repo add k8s-external-secrets https://external-secrets.github.io/kubernetes-external-secrets/

$ helm install external-secrets external-secrets/external-secrets \
    --set serviceAccount.annotations."iam\.gke\.io/gcp-service-account"='secret-gsa@'"${PROJECT_ID}"'.iam.gserviceaccount.com' \
    --set serviceAccount.create=true \
    --set serviceAccount.name="secret-ksa"

helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets/external-secrets

helm install external-secrets external-secrets/external-secrets \
    --set serviceAccount.annotations."iam\.gke\.io/gcp-service-account"='secret-gsa@gke-practice-341914.iam.gserviceaccount.com' \
    --set serviceAccount.create=true \
    --set serviceAccount.name="secret-ksa"

helm uninstall external-secrets
```

Verify that controller and CRDs have been installed

```bash
kubectl get po -l app.kubernetes.io/name=external-secrets
```

You should see one pod

```bash
NAME                                                            READY   STATUS    RESTARTS   AGE
external-secrets-kubernetes-external-secrets-694f98f7bhbxgp   1/1     Running       0          20m
```

```bash
kubectl get crd | grep external
```

You should see 1 crd

```bash
externalsecrets.kubernetes-client.io                        2021-11-18T10:53:05Z
```

## Configuring External Secrets

Create a Secret in GSM. NB: External Secrets requires secrets to be in JSON format, we will explain why later.

```bash
echo -n '{"value": "my-secret-value"}' | gcloud secrets create my-secret --replication-policy="automatic" --data-file=-
```

Grant the GSA access to Secrets. NB: We grant access to ALL Secrets in the project as External Secrets doesn't support fine grain access, the controller in the cluster would access secrets in GSM on behalf of all workloads that needs one.

```bash
$ gcloud config set project $PROJECT_ID
gcloud config set project gke-practice-341914

gcloud projects add-iam-policy-binding ${PROJECT_ID} \
    --member=serviceAccount:secret-gsa@${PROJECT_ID}.iam.gserviceaccount.com \
    --role=roles/secretmanager.secretAccessor

gcloud projects add-iam-policy-binding gke-practice-341914 `
    --member=serviceAccount:secret-gsa@gke-practice-341914.iam.gserviceaccount.com `
    --role=roles/secretmanager.secretAccessor
```
